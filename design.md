# MeetMind — Design Document

## Overview

MeetMind is a mobile-first AI meeting intelligence platform. Users record meetings via their phone
microphone; audio is streamed in real time to a FastAPI backend, transcribed by OpenAI Whisper,
and then processed by an LLM (GPT-4o / Claude 3.5) to produce structured summaries, decisions,
and action items. A RAG pipeline over pgvector enables natural-language queries against all past
meetings. The system operates on a freemium model enforced at the API layer.

This document covers the complete technical design: system architecture, component responsibilities,
data models, API contracts, WebSocket protocol, AI pipeline internals, frontend architecture,
security, infrastructure, error handling, and testing strategy.

---

## Architecture

### System Component Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     React Native App (iOS / Android)                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ Zustand  │  │  React   │  │  Expo    │  │  Expo            │   │
│  │  Store   │  │  Query   │  │  Audio   │  │  Notifications   │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
│       │              │              │                  │             │
│  ┌────▼──────────────▼──────────────▼──────────────────▼─────────┐ │
│  │              Navigation Layer (React Navigation v6)            │ │
│  └────────────────────────────┬───────────────────────────────────┘ │
└───────────────────────────────┼─────────────────────────────────────┘
                                │
              ┌─────────────────┴──────────────────┐
              │  REST (HTTPS/Axios)   WS (WSS)      │
              │  SSE (AI Chat)                      │
              └─────────────────┬──────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────┐
│                        FastAPI Backend                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ Auth_Service │  │Meeting_Svc   │  │  WebSocket Handler       │  │
│  │ (JWT/bcrypt) │  │Task_Service  │  │  /ws/transcribe/{id}     │  │
│  │              │  │Mood_Service  │  └──────────┬───────────────┘  │
│  └──────┬───────┘  │Analytics_Svc │             │                  │
│         │          │Notif_Service │  ┌──────────▼───────────────┐  │
│         │          │Integr_Svc    │  │  Transcription_Service   │  │
│         │          └──────┬───────┘  │  (Whisper)               │  │
│         │                 │          └──────────────────────────┘  │
│  ┌──────▼─────────────────▼──────────────────────────────────────┐ │
│  │              SQLAlchemy 2.0 ORM  +  Alembic Migrations        │ │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
         │                          │
         ▼                          ▼
┌─────────────────┐      ┌──────────────────────────────────────────┐
│  PostgreSQL 16  │      │           Redis 7                        │
│  + pgvector     │      │  Celery Broker  │  Rate Limit Counters   │
└─────────────────┘      └──────────────────────────────────────────┘
                                    │
                         ┌──────────▼──────────────────────────────┐
                         │         Celery Worker Pool               │
                         │  ┌─────────────────────────────────┐    │
                         │  │  Summarization_Service (GPT-4o) │    │
                         │  │  Embedding_Service (embed-3-sm) │    │
                         │  └─────────────────────────────────┘    │
                         └─────────────────────────────────────────┘
                                    │
                         ┌──────────▼──────────────────────────────┐
                         │         Cloudflare R2                    │
                         │  (temporary audio buffer — deleted       │
                         │   immediately after transcription)       │
                         └─────────────────────────────────────────┘
```

### Communication Channels

| Channel | Protocol | Used For |
|---------|----------|----------|
| REST API | HTTPS (TLS 1.2+) | Auth, CRUD, analytics, task updates, mood, integrations |
| WebSocket | WSS (TLS 1.2+) | Live audio chunk streaming + real-time transcript delivery |
| SSE | HTTPS chunked | AI chat token streaming from RAG endpoint |
| Celery | Redis AMQP | Async LLM summarization + embedding generation |

### Deployment Topology

```
Internet
   │
   ▼
Cloudflare (DNS + TLS termination)
   │
   ├──► Railway/Render — FastAPI (Uvicorn, 2+ workers)
   │         │
   │         ├──► PostgreSQL 16 (managed, Railway)
   │         ├──► Redis 7 (managed, Railway)
   │         └──► Celery Worker (separate Railway service)
   │
   └──► Cloudflare R2 (audio buffer, auto-deleted)

GitHub Actions CI/CD:
  push → test → build Docker image → deploy to Railway
```

---

## Components and Interfaces

### Auth_Service

Responsibilities: user registration, login, Google OAuth, JWT issuance, token refresh, password hashing.

Key interfaces:
- `register(name, email, password) → UserOut + JWT`
- `login(email, password) → TokenPair`
- `google_oauth(code) → TokenPair`
- `refresh(refresh_token) → AccessToken`
- `verify_token(token) → UserClaims`

Implementation notes:
- Passwords hashed with `passlib[bcrypt]`, cost factor 12 (NFR-1.1)
- JWTs signed with HS256, 256-bit secret from env (NFR-1.2)
- Access token TTL: 60 minutes; Refresh token TTL: 30 days (Req 2, AC4/5)
- Refresh tokens stored in `refresh_tokens` table (hashed) for revocation support
- Google OAuth: exchange code via `google-auth` library, upsert user record

### Meeting_Service

Responsibilities: meeting lifecycle (start, stop, list, retrieve, delete), plan enforcement delegation.

Key interfaces:
- `start_meeting(user_id, title?) → Meeting`
- `stop_meeting(meeting_id, user_id) → Meeting` (enqueues Celery task)
- `list_meetings(user_id, search?, date_from?, date_to?, page, size) → Page[MeetingOut]`
- `get_meeting(meeting_id, user_id) → MeetingDetail`
- `delete_meeting(meeting_id, user_id) → None` (cascades all related data)

Plan enforcement (delegated to Plan_Enforcer):
- Free: max 5 meetings/month, max 30 min/meeting, 7-day history window
- Pro: unlimited meetings, 1-year history
- Team: all Pro features + shared board

### Transcription_Service

Responsibilities: receive audio chunks over WebSocket, run Whisper inference, return partial transcript.

Key interfaces:
- `transcribe_chunk(audio_bytes: bytes, meeting_id: UUID) → str`
- Whisper model loaded once at startup (base model for speed, large-v3 for accuracy on Pro)
- Each chunk processed independently; failures logged and skipped (Req 27, AC3)
- Target: < 2 seconds per 30-second chunk (Req 28, AC1)

### Summarization_Service

Responsibilities: build LLM prompt from full transcript, call GPT-4o, parse structured JSON response, persist summary.

Key interfaces:
- `summarize(meeting_id: UUID, transcript: str) → SummaryOut`
- Retry logic: up to 3 total attempts with exponential backoff (Req 7, AC5)
- Validates response schema before persisting (Req 7, AC4)
- On success: updates meeting status to "done", triggers Embedding_Service

### Embedding_Service

Responsibilities: chunk transcript text, generate embeddings via OpenAI, store in pgvector.

Key interfaces:
- `embed_transcript(transcript_id: UUID, full_text: str) → List[EmbeddingRecord]`
- Chunking: ~500 tokens per chunk, 50-token overlap (Req 19, AC1)
- Model: `text-embedding-3-small` → 1536-dim vectors (Req 19, AC2)
- Retry per chunk: up to 3 attempts with exponential backoff (Req 19, AC4)
- Runs asynchronously via Celery after summarization (Req 19, AC7)

### RAG_Service

Responsibilities: encode user question, cosine similarity search in pgvector, build context prompt, stream LLM response via SSE.

Key interfaces:
- `ask(user_id: UUID, question: str) → AsyncGenerator[str]`
- Top-k retrieval: k=5 most similar chunks (configurable)
- Uses same `text-embedding-3-small` model for question encoding (Req 18, AC7)
- If no chunks found: returns "no relevant content" message (Req 18, AC5)
- Streams tokens via SSE using FastAPI `StreamingResponse`

### Task_Service

Responsibilities: CRUD for tasks, status updates, Kanban grouping.

Key interfaces:
- `list_tasks(user_id, status?) → List[TaskOut]`
- `update_status(task_id, user_id, status) → TaskOut`
- `create_tasks_from_summary(meeting_id, tasks_json) → List[Task]`

### Mood_Service

Responsibilities: record mood scores, aggregate heatmap data.

Key interfaces:
- `submit_mood(user_id, meeting_id, score: int) → MoodOut` (upsert)
- `get_heatmap(team_id, date_from, date_to) → List[DayMood]`
- Score validation: 1 ≤ score ≤ 5 (Req 14, AC3)

### Analytics_Service

Responsibilities: aggregate weekly stats for dashboard charts.

Key interfaces:
- `get_summary(user_id, week_start) → AnalyticsSummary`
- Returns: meetings_count, total_hours, avg_sentiment, daily_breakdown

### Plan_Enforcer

Responsibilities: centralised plan limit checks called by other services before operations.

Key interfaces:
- `check_meeting_start(user_id) → None | raises HTTP403`
- `check_duration(user_id, elapsed_seconds) → None | raises HTTP403`
- `check_feature_access(user_id, feature: Feature) → None | raises HTTP403`
- `check_history_window(user_id) → date` (returns earliest allowed date)
- Reads user.plan from DB; enforcement is real-time (Req 21, AC7)

### Notification_Service

Responsibilities: schedule and cancel push notifications via Expo Push API.

Key interfaces:
- `schedule_pre_meeting_brief(meeting_id, start_time, user_expo_token) → None`
- `cancel_notification(meeting_id) → None`
- Respects user notification preference from settings (Req 23, AC3)

### Integration_Service

Responsibilities: format and export meeting summaries to external platforms.

Key interfaces:
- `export(meeting_id, user_id, platform: Platform) → ExportResult`
- Platforms: Slack (Incoming Webhooks), Notion (API), Trello (API), Email (SMTP)
- Pro/Team plan only (Req 20, AC8)

---

## Data Models

### PostgreSQL Schema (SQLAlchemy 2.0 / Alembic)

```sql
-- users
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255),          -- NULL for OAuth-only accounts
    google_id   VARCHAR(255) UNIQUE,     -- NULL for password accounts
    plan        VARCHAR(20) NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'pro', 'team')),
    team_id     UUID REFERENCES teams(id) ON DELETE SET NULL,
    is_team_admin BOOLEAN NOT NULL DEFAULT FALSE,
    expo_push_token VARCHAR(255),
    notify_enabled  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_team_id ON users(team_id);

-- teams
CREATE TABLE teams (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id    UUID NOT NULL REFERENCES users(id),
    invite_token VARCHAR(64) UNIQUE,
    invite_expires_at TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- refresh_tokens
CREATE TABLE refresh_tokens (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  VARCHAR(255) NOT NULL UNIQUE,
    expires_at  TIMESTAMPTZ NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);

-- meetings
CREATE TABLE meetings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL DEFAULT 'Untitled Meeting',
    status          VARCHAR(20) NOT NULL DEFAULT 'recording'
                        CHECK (status IN ('recording','processing','done','failed')),
    duration_seconds INTEGER,
    scheduled_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_meetings_user_id ON meetings(user_id);
CREATE INDEX idx_meetings_created_at ON meetings(created_at DESC);
CREATE INDEX idx_meetings_status ON meetings(status);

-- transcripts
CREATE TABLE transcripts (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id  UUID NOT NULL UNIQUE REFERENCES meetings(id) ON DELETE CASCADE,
    full_text   TEXT NOT NULL DEFAULT '',
    chunks_json JSONB NOT NULL DEFAULT '[]',  -- [{seq, text, start_ms, end_ms}]
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- summaries
CREATE TABLE summaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL UNIQUE REFERENCES meetings(id) ON DELETE CASCADE,
    summary         TEXT NOT NULL,
    decisions_json  JSONB NOT NULL DEFAULT '[]',  -- [string]
    tasks_json      JSONB NOT NULL DEFAULT '[]',  -- [{text, assignee, due_date}]
    raw_llm_response TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- tasks
CREATE TABLE tasks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id  UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    text        VARCHAR(500) NOT NULL,
    assignee    VARCHAR(255),
    status      VARCHAR(20) NOT NULL DEFAULT 'todo'
                    CHECK (status IN ('todo','in_progress','done')),
    due_date    DATE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_tasks_meeting_id ON tasks(meeting_id);
CREATE INDEX idx_tasks_status ON tasks(status);

-- moods
CREATE TABLE moods (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id  UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    score       SMALLINT NOT NULL CHECK (score >= 1 AND score <= 5),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (meeting_id, user_id)  -- one mood per user per meeting (upsert)
);
CREATE INDEX idx_moods_user_id ON moods(user_id);
CREATE INDEX idx_moods_created_at ON moods(created_at);

-- embeddings (pgvector)
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transcript_id   UUID NOT NULL REFERENCES transcripts(id) ON DELETE CASCADE,
    chunk_seq       INTEGER NOT NULL,
    chunk_text      TEXT NOT NULL,
    vector          vector(1536) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_embeddings_transcript_id ON embeddings(transcript_id);
-- IVFFlat index for approximate nearest-neighbour search
CREATE INDEX idx_embeddings_vector ON embeddings
    USING ivfflat (vector vector_cosine_ops) WITH (lists = 100);
```

### Pydantic v2 Request / Response Models

```python
# auth
class RegisterRequest(BaseModel):
    name: str = Field(min_length=1, max_length=255)
    email: EmailStr
    password: str = Field(min_length=8)

class LoginRequest(BaseModel):
    email: EmailStr
    password: str

class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

class UserOut(BaseModel):
    id: UUID
    name: str
    email: EmailStr
    plan: Literal["free", "pro", "team"]
    created_at: datetime

# meetings
class MeetingCreate(BaseModel):
    title: str = Field(default="Untitled Meeting", max_length=500)

class MeetingOut(BaseModel):
    id: UUID
    title: str
    status: Literal["recording", "processing", "done", "failed"]
    duration_seconds: int | None
    created_at: datetime

class MeetingDetail(MeetingOut):
    transcript: str | None
    summary: str | None
    decisions: list[str]
    tasks: list[TaskOut]

# tasks
class TaskOut(BaseModel):
    id: UUID
    meeting_id: UUID
    text: str
    assignee: str | None
    status: Literal["todo", "in_progress", "done"]
    due_date: date | None
    source_meeting_title: str

class TaskStatusUpdate(BaseModel):
    status: Literal["todo", "in_progress", "done"]

# mood
class MoodSubmit(BaseModel):
    meeting_id: UUID
    score: int = Field(ge=1, le=5)

class DayMood(BaseModel):
    date: date
    avg_score: float | None  # null if no submissions

# analytics
class AnalyticsSummary(BaseModel):
    week_start: date
    meetings_count: int
    total_hours: float
    avg_sentiment: float | None
    daily_meetings: list[DailyCount]  # [{date, count}]

# AI chat
class AskRequest(BaseModel):
    question: str = Field(min_length=1, max_length=2000)

# embeddings (internal)
class ChunkRecord(BaseModel):
    seq: int
    text: str
    start_ms: int
    end_ms: int
```

---

## API Design

### Authentication Endpoints

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | /auth/register | Public | `RegisterRequest` | `TokenPair + UserOut` |
| POST | /auth/login | Public | `LoginRequest` | `TokenPair` |
| POST | /auth/google | Public | `{code: str}` | `TokenPair` |
| POST | /auth/refresh | Public | `{refresh_token: str}` | `{access_token: str}` |
| GET | /auth/me | Bearer | — | `UserOut` |
| PATCH | /auth/me | Bearer | `{name: str}` | `UserOut` |
| POST | /auth/logout | Bearer | — | `204` |

Rate limit: 10 req/min per IP on register, login, google (Req 26, AC3).

### Meeting Endpoints

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | /meetings/start | Bearer | `MeetingCreate` | `MeetingOut` |
| POST | /meetings/{id}/stop | Bearer | — | `MeetingOut` |
| GET | /meetings | Bearer | `?search=&date_from=&date_to=&page=&size=` | `Page[MeetingOut]` |
| GET | /meetings/{id} | Bearer | — | `MeetingDetail` |
| DELETE | /meetings/{id} | Bearer | — | `204` |

### Task Endpoints

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| GET | /tasks | Bearer | `?status=todo\|in_progress\|done` | `List[TaskOut]` |
| POST | /tasks/{id}/status | Bearer | `TaskStatusUpdate` | `TaskOut` |

### Mood Endpoints

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | /mood | Bearer | `MoodSubmit` | `MoodOut` |
| GET | /mood/heatmap | Bearer (Team) | `?date_from=&date_to=` | `List[DayMood]` |

### Analytics Endpoint

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| GET | /analytics/summary | Bearer | `?week_start=YYYY-MM-DD` | `AnalyticsSummary` |

### AI Chat Endpoint

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | /ai/ask | Bearer (Pro/Team) | `AskRequest` | `text/event-stream` (SSE) |

SSE event format:
```
data: {"token": "The budget"}\n\n
data: {"token": " was approved"}\n\n
data: {"done": true, "sources": [{"meeting_id": "...", "title": "..."}]}\n\n
```

### WebSocket Endpoint

```
WS /ws/transcribe/{meeting_id}
Headers: Authorization: Bearer <token>
```

See WebSocket Protocol section below.

### Infrastructure Endpoints

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | /health | Public | `{"status":"ok","db":"ok","redis":"ok","celery":"ok"}` |
| GET | /metrics | Internal | Prometheus-format metrics |

---

## WebSocket Protocol Design

### Connection Lifecycle

```
Client                                    Server
  │                                          │
  │── WSS /ws/transcribe/{meeting_id} ──────►│
  │   Header: Authorization: Bearer <jwt>    │
  │                                          │── validate JWT + meeting ownership
  │                                          │── load meeting record
  │◄── {"type":"connected","meeting_id":"…"} │
  │                                          │
  │── binary frame (AAC chunk, ~30s) ───────►│
  │                                          │── Whisper.transcribe(chunk)
  │◄── {"type":"transcript","seq":1,         │
  │     "text":"Hello everyone…","ms":30000} │
  │                                          │
  │── binary frame (AAC chunk) ─────────────►│
  │◄── {"type":"transcript","seq":2,…}       │
  │                                          │
  │── {"type":"stop"} ──────────────────────►│
  │                                          │── update meeting status → "processing"
  │                                          │── enqueue Celery summarization task
  │◄── {"type":"stopped","duration_ms":…}    │
  │                                          │── close connection
```

### Message Schema

**Client → Server (binary):** Raw AAC audio bytes, one chunk per frame.

**Client → Server (JSON control):**
```json
{"type": "stop"}
{"type": "ping"}
{"type": "chunk_ack", "seq": 3}  // optional flow control
```

**Server → Client (JSON):**
```json
{"type": "connected", "meeting_id": "uuid"}
{"type": "transcript", "seq": 1, "text": "...", "start_ms": 0, "end_ms": 30000}
{"type": "error", "code": "WHISPER_FAILED", "seq": 2, "message": "..."}
{"type": "stopped", "duration_ms": 1800000}
{"type": "pong"}
```

### Chunk Sequencing and Buffering

- Client assigns monotonically increasing `seq` numbers to chunks (tracked in app state)
- Server processes chunks in order; out-of-order chunks are queued by seq
- On reconnect, client resends all unacknowledged chunks starting from last acked seq
- Server deduplicates by (meeting_id, seq) — if seq already processed, returns cached transcript (P22)

### Reconnection Backoff Algorithm

```
attempt = 0
delay = 1s
max_delay = 30s

while not connected:
    wait(delay)
    try connect()
    if connected:
        flush_buffer_in_order()
        break
    attempt += 1
    delay = min(delay * 2, max_delay)  // exponential backoff
```

Matches Req 25, AC4: 1s → 2s → 4s → 8s → 16s → 30s (capped).

---

## AI Pipeline Design

### Whisper Transcription Pipeline

```
Audio Chunk (AAC, ~30s)
        │
        ▼
[Decode AAC → PCM 16kHz mono]  (ffmpeg via subprocess or pydub)
        │
        ▼
[whisper.transcribe(audio_array, language="auto")]
        │
        ▼
[Extract .text field]
        │
        ▼
[Append to transcript.chunks_json with seq + timestamps]
        │
        ▼
[WebSocket send: {"type":"transcript","seq":N,"text":"..."}]
```

Model selection:
- Default: `whisper-base` (fast, ~1s for 30s chunk on CPU)
- Pro plan: `whisper-large-v3` via OpenAI API (higher accuracy, ~1.5s)
- Offline fallback: `whisper-base` bundled in React Native via `whisper.rn`

### LLM Summarization Prompt

```python
SYSTEM_PROMPT = """You are a meeting intelligence assistant. 
Analyze the provided meeting transcript and return a JSON object with exactly these fields:
- "summary": A concise paragraph (3-5 sentences) summarizing the key discussion points.
- "decisions": An array of strings, each describing a decision made during the meeting.
  Return an empty array if no decisions were made.
- "tasks": An array of objects with fields:
    - "text": The action item description (max 500 chars)
    - "assignee": The person's name as mentioned in the transcript, or null if unspecified
    - "due_date": ISO date string if mentioned, or null
  Return an empty array if no tasks were identified.
Only include assignees whose names appear verbatim or as clear references in the transcript.
Return ONLY valid JSON. No markdown, no explanation."""

USER_PROMPT = f"""Meeting transcript:
{transcript_text}

Return the JSON analysis:"""
```

Response validation (Req 7, AC4):
```python
class SummaryJSON(BaseModel):
    summary: str = Field(min_length=1)
    decisions: list[str]
    tasks: list[TaskJSON]

class TaskJSON(BaseModel):
    text: str = Field(min_length=1, max_length=500)
    assignee: str | None
    due_date: date | None
```

Retry logic: up to 3 attempts. On final failure: set meeting.status = "failed".

### Embedding Pipeline

```
Full Transcript Text
        │
        ▼
[Token-aware chunker: ~500 tokens, 50-token overlap]
  Uses tiktoken cl100k_base tokenizer
        │
        ▼
[For each chunk in parallel (asyncio.gather)]
        │
        ▼
[openai.embeddings.create(model="text-embedding-3-small", input=chunk_text)]
        │
        ▼
[Assert len(vector) == 1536]
        │
        ▼
[INSERT INTO embeddings (transcript_id, chunk_seq, chunk_text, vector)]
        │
        ▼
[IVFFlat index auto-updated by pgvector]
```

Chunking algorithm:
```python
def chunk_transcript(text: str, max_tokens=500, overlap=50) -> list[str]:
    enc = tiktoken.get_encoding("cl100k_base")
    tokens = enc.encode(text)
    chunks = []
    start = 0
    while start < len(tokens):
        end = min(start + max_tokens, len(tokens))
        chunk_tokens = tokens[start:end]
        chunks.append(enc.decode(chunk_tokens))
        start += max_tokens - overlap
    return chunks
```

### RAG Query Flow

```
User Question
        │
        ▼
[embed(question, model="text-embedding-3-small")]  → 1536-dim vector
        │
        ▼
[pgvector cosine similarity search]
  SELECT chunk_text, transcript_id, 1 - (vector <=> $query_vec) AS score
  FROM embeddings
  JOIN transcripts ON embeddings.transcript_id = transcripts.id
  JOIN meetings ON transcripts.meeting_id = meetings.id
  WHERE meetings.user_id = $user_id
  ORDER BY vector <=> $query_vec
  LIMIT 5
        │
        ▼
[If no results: return "no relevant content" message]
        │
        ▼
[Build context prompt with retrieved chunks + source citations]
        │
        ▼
[GPT-4o streaming call with context]
        │
        ▼
[Stream tokens via SSE to client]
        │
        ▼
[On stream complete: send sources array]
```

---

## Frontend Architecture

### Screen Navigation Structure

```
Root Navigator (Stack)
├── Onboarding Stack (unauthenticated)
│   └── S01 — Onboarding (register / Google sign-in)
│
└── Main App (authenticated)
    ├── Bottom Tab Navigator
    │   ├── Tab: Home → S02 Home Dashboard
    │   ├── Tab: Tasks → S06 Task Board
    │   ├── Tab: Analytics → S07 Analytics
    │   ├── Tab: Mood → S08 Team Mood
    │   └── Tab: Settings → S11 Settings
    │
    └── Modal / Stack Screens (pushed on top of tabs)
        ├── S03 — Live Recording (full-screen modal)
        ├── S04 — Meeting Summary (stack push)
        ├── S05 — Meeting History (stack push from Home)
        └── S09 — AI Chat (stack push)
        └── S10 — Integrations (stack push from S04)
```

### Zustand Store Design

```typescript
// auth store
interface AuthStore {
  user: UserOut | null;
  accessToken: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
  login: (tokens: TokenPair, user: UserOut) => void;
  logout: () => void;
  updateTokens: (accessToken: string) => void;
}

// meeting store (live recording state — not persisted)
interface MeetingStore {
  activeMeetingId: string | null;
  isRecording: boolean;
  liveTranscript: string;
  chunkBuffer: AudioChunk[];  // buffered during WS disconnect
  wsStatus: 'connected' | 'disconnected' | 'reconnecting';
  reconnectAttempt: number;
  appendTranscript: (text: string) => void;
  bufferChunk: (chunk: AudioChunk) => void;
  clearBuffer: () => void;
  setWsStatus: (status: WsStatus) => void;
}

// ui store
interface UIStore {
  isOffline: boolean;
  setOffline: (v: boolean) => void;
}
```

Persistence: `auth` store persisted to `SecureStore` via a custom Zustand middleware.
`meeting` store is ephemeral (in-memory only).

### React Query Patterns

```typescript
// meetings list — cached, background refetch on focus
const { data, isLoading } = useQuery({
  queryKey: ['meetings', filters],
  queryFn: () => api.getMeetings(filters),
  staleTime: 30_000,
});

// meeting detail — cached per meeting_id
const { data } = useQuery({
  queryKey: ['meeting', meetingId],
  queryFn: () => api.getMeeting(meetingId),
  // poll while status is 'processing'
  refetchInterval: (data) =>
    data?.status === 'processing' ? 3000 : false,
});

// task status update — optimistic
const mutation = useMutation({
  mutationFn: ({ taskId, status }) => api.updateTaskStatus(taskId, status),
  onMutate: async ({ taskId, status }) => {
    await queryClient.cancelQueries({ queryKey: ['tasks'] });
    const prev = queryClient.getQueryData(['tasks']);
    queryClient.setQueryData(['tasks'], (old) =>
      old.map(t => t.id === taskId ? { ...t, status } : t)
    );
    return { prev };
  },
  onError: (_, __, ctx) => queryClient.setQueryData(['tasks'], ctx.prev),
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['tasks'] }),
});
```

### Axios Interceptor (JWT Auto-Attach + Refresh)

```typescript
// request interceptor
api.interceptors.request.use(async (config) => {
  const token = useAuthStore.getState().accessToken;
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// response interceptor — handle 401 with token refresh
api.interceptors.response.use(
  (res) => res,
  async (error) => {
    if (error.response?.status === 401 && !error.config._retry) {
      error.config._retry = true;
      const refreshToken = useAuthStore.getState().refreshToken;
      if (!refreshToken) { logout(); return Promise.reject(error); }
      try {
        const { data } = await axios.post('/auth/refresh', { refresh_token: refreshToken });
        useAuthStore.getState().updateTokens(data.access_token);
        error.config.headers.Authorization = `Bearer ${data.access_token}`;
        return api(error.config);
      } catch {
        logout();
        return Promise.reject(error);
      }
    }
    return Promise.reject(error);
  }
);
```

---

## Security Design

### JWT Flow

```
1. Client sends credentials → Auth_Service
2. Auth_Service verifies password (bcrypt.verify)
3. Auth_Service creates:
   - access_token: JWT {sub: user_id, plan: "free", exp: now+60min}, signed HS256
   - refresh_token: random 256-bit token, stored as SHA-256 hash in refresh_tokens table
4. Client stores both in SecureStore (encrypted, hardware-backed on supported devices)
5. Every request: Axios attaches "Authorization: Bearer <access_token>"
6. FastAPI dependency: decode JWT → extract user_id → load user from DB
7. On 401: Axios interceptor calls /auth/refresh with refresh_token
8. On logout: DELETE refresh_token from DB + clear SecureStore
```

### Password Security

- bcrypt cost factor 12 (NFR-1.1) — ~250ms hash time, brute-force resistant
- Minimum 8 characters enforced at Pydantic layer (Req 1, AC3)
- Login errors return identical 401 for wrong password and unknown email (Req 2, AC3) — prevents user enumeration

### Transport Security

- All REST: HTTPS with TLS 1.2+ enforced at Cloudflare edge (NFR-1.5)
- All WebSocket: WSS with TLS 1.2+ (Req 24, AC4)
- HTTP → HTTPS redirect at Cloudflare
- HSTS header set on all responses

### Input Sanitization

- All string inputs validated by Pydantic v2 (type coercion + length limits)
- SQLAlchemy parameterized queries — no raw SQL string interpolation
- HTML/JS in user strings escaped at render layer in React Native

### SecureStore Usage

```typescript
// write
await SecureStore.setItemAsync('access_token', token, {
  keychainAccessible: SecureStore.WHEN_UNLOCKED
});
// read
const token = await SecureStore.getItemAsync('access_token');
// delete on logout
await SecureStore.deleteItemAsync('access_token');
await SecureStore.deleteItemAsync('refresh_token');
```

---

## Infrastructure Design

### Docker Compose (Development)

```yaml
version: "3.9"
services:
  api:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://mm:mm@db:5432/meetmind
      REDIS_URL: redis://redis:6379/0
      JWT_SECRET: ${JWT_SECRET}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      R2_BUCKET: ${R2_BUCKET}
      R2_ACCESS_KEY: ${R2_ACCESS_KEY}
      R2_SECRET_KEY: ${R2_SECRET_KEY}
    depends_on: [db, redis]
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  worker:
    build: ./backend
    environment:
      DATABASE_URL: postgresql+asyncpg://mm:mm@db:5432/meetmind
      REDIS_URL: redis://redis:6379/0
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    depends_on: [db, redis]
    command: celery -A app.worker worker --loglevel=info --concurrency=4

  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: mm
      POSTGRES_PASSWORD: mm
      POSTGRES_DB: meetmind
    volumes: ["pgdata:/var/lib/postgresql/data"]
    ports: ["5432:5432"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

### Production Deployment (Railway)

- `api` service: FastAPI + Uvicorn, 2 workers, auto-scaled
- `worker` service: Celery, 4 concurrent workers, separate Railway service
- `db`: Railway managed PostgreSQL 16 with pgvector extension enabled
- `redis`: Railway managed Redis 7
- Environment variables: set in Railway dashboard, never in code
- GitHub Actions: on push to `main` → run tests → build Docker → deploy to Railway

### GitHub Actions CI/CD

```yaml
# .github/workflows/ci.yml
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env: {POSTGRES_PASSWORD: test}
      redis:
        image: redis:7-alpine
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.12"}
      - run: pip install -r backend/requirements.txt
      - run: pytest backend/tests/ --cov=app --cov-report=xml
      - uses: codecov/codecov-action@v4

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: railway up --service api
        env: {RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}}
```

---

## Error Handling Patterns

### Backend Error Hierarchy

```python
class MeetMindError(Exception):
    status_code: int = 500
    code: str = "INTERNAL_ERROR"
    message: str = "An unexpected error occurred"

class AuthError(MeetMindError):
    status_code = 401; code = "AUTH_ERROR"

class ForbiddenError(MeetMindError):
    status_code = 403; code = "FORBIDDEN"

class NotFoundError(MeetMindError):
    status_code = 404; code = "NOT_FOUND"

class ConflictError(MeetMindError):
    status_code = 409; code = "CONFLICT"

class PlanLimitError(ForbiddenError):
    code = "PLAN_LIMIT_EXCEEDED"

class RateLimitError(MeetMindError):
    status_code = 429; code = "RATE_LIMITED"
```

Global exception handler:
```python
@app.exception_handler(MeetMindError)
async def meetmind_error_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.code, "message": exc.message,
                 "request_id": request.state.request_id}
    )

@app.exception_handler(Exception)
async def generic_error_handler(request, exc):
    logger.error("Unhandled error", exc_info=exc,
                 extra={"request_id": request.state.request_id})
    return JSONResponse(status_code=500,
        content={"error": "INTERNAL_ERROR",
                 "message": "An unexpected error occurred"})
```

### Frontend Error Handling

```typescript
// React Query global error handler
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        if (error.response?.status === 401) return false;  // don't retry auth errors
        if (error.response?.status === 403) return false;  // don't retry plan errors
        return failureCount < 2;
      },
    },
  },
});

// Offline detection
NetInfo.addEventListener(state => {
  useUIStore.getState().setOffline(!state.isConnected);
});
```

### Celery Task Error Handling

```python
@celery_app.task(bind=True, max_retries=3, default_retry_delay=5)
async def summarize_meeting(self, meeting_id: str):
    try:
        # ... summarization logic
    except LLMResponseError as exc:
        if self.request.retries < self.max_retries:
            raise self.retry(exc=exc, countdown=2 ** self.request.retries)
        # Final failure: mark meeting as failed
        await meeting_service.mark_failed(meeting_id)
        await notify_user_of_failure(meeting_id)
```

### Rate Limiting Implementation

```python
# Redis sliding window rate limiter
async def check_rate_limit(redis: Redis, key: str, limit: int, window_seconds: int):
    now = time.time()
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, now - window_seconds)
    pipe.zadd(key, {str(now): now})
    pipe.zcard(key)
    pipe.expire(key, window_seconds)
    results = await pipe.execute()
    count = results[2]
    if count > limit:
        raise RateLimitError(retry_after=window_seconds)
```

---
## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Registration Round-Trip

*For any* valid registration input (non-empty name, valid RFC 5322 email, password of length ≥ 8), calling `register(name, email, password)` returns a JWT that is accepted by `GET /meetings` without a 401 error.

**Validates: Requirements 1.7**

---

### Property 2: Login Round-Trip

*For any* registered user, the full sequence `login(email, password)` → `GET /meetings` with access token → `POST /auth/refresh` → `GET /meetings` with new access token succeeds at every step without re-entering credentials.

**Validates: Requirements 2.8**

---

### Property 3: Meeting ID Uniqueness

*For any* sequence of N meeting start requests by any set of authenticated users, all returned `meeting_id` values are distinct — no two meetings share the same identifier.

**Validates: Requirements 5.5**

---

### Property 4: Transcript Completeness

*For any* sequence of audio chunks C1, C2, ..., Cn transmitted for a meeting, the concatenation of all partial transcript texts returned by the Transcription_Service equals the `full_text` stored in the transcripts table for that meeting.

**Validates: Requirements 6.6**

---

### Property 5: Summary JSON Structure Invariant

*For any* valid transcript T, `summarize(T)` always returns a JSON object containing: a non-null `summary` string, a `decisions` array (possibly empty), and a `tasks` array (possibly empty). The response is never null, never missing a field, and never returns a non-array for decisions or tasks.

**Validates: Requirements 7.4, 9.4, 10.2**

---

### Property 6: Task Grounding Invariant

*For any* valid transcript T and the extracted task set `tasks = summarize(T).tasks`, for each task where `task.assignee` is non-null, the assignee string appears as a substring or clear reference within T.

**Validates: Requirements 7.7, 9.6**

---

### Property 7: Search Subset Invariant

*For any* search query Q and authenticated user U, the result set of `search(U, Q)` is always a subset of `all_meetings(U)` — no meeting appears in search results that does not belong to the user's full meeting list.

**Validates: Requirements 11.7**

---

### Property 8: Date Filter Correctness

*For any* date range [start, end] and authenticated user U, every meeting M returned by `filter_by_date(U, start, end)` satisfies `start ≤ M.created_at ≤ end`.

**Validates: Requirements 11.8**

---

### Property 9: Deletion Cascade Invariant

*For any* meeting M that is deleted, subsequent queries for transcripts, summaries, tasks, moods, and embeddings associated with `M.id` all return empty result sets — no orphaned records remain in any table.

**Validates: Requirements 12.1, 12.5, 12.6**

---

### Property 10: Kanban Partition Invariant

*For any* authenticated user U, the sum `count(tasks_todo(U)) + count(tasks_in_progress(U)) + count(tasks_done(U))` always equals `count(all_tasks(U))` — every task belongs to exactly one status column.

**Validates: Requirements 13.5**

---

### Property 11: Task Status Update Idempotence

*For any* task T with current status S, calling `update_status(T, S)` (updating to the same status) followed by `get_task(T)` returns a task with status S, identical to the result before the update — the operation is a no-op.

**Validates: Requirements 13.7**

---

### Property 12: Mood Score Range Invariant

*For all* Mood_Score records M stored in the database, `M.score` is an integer satisfying `1 ≤ M.score ≤ 5`. No score outside this closed interval can be persisted.

**Validates: Requirements 14.6**

---

### Property 13: Analytics Meetings Count Invariant

*For any* user U and time period P, `analytics(U, P).meetings_count` equals the count of meetings where `user_id = U`, `status = 'done'`, and `created_at` falls within P.

**Validates: Requirements 16.6**

---

### Property 14: Analytics Hours Calculation Invariant

*For any* user U and time period P, `analytics(U, P).total_hours` equals `sum(M.duration_seconds for M in meetings(U, P)) / 3600`, rounded to two decimal places.

**Validates: Requirements 16.7**

---

### Property 15: Embedding Dimension Invariant

*For all* inputs to the `text-embedding-3-small` model — whether transcript chunks stored in pgvector or user questions submitted to the RAG_Service — the resulting vector has exactly 1536 dimensions. This ensures all vectors occupy the same vector space for cosine similarity search.

**Validates: Requirements 18.7, 18.8, 19.5**

---

### Property 16: Embedding Round-Trip Retrieval

*For any* transcript chunk C that has been embedded and stored in pgvector, performing a nearest-neighbour cosine similarity search using `embed(C)` as the query vector returns C as the top result.

**Validates: Requirements 18.9**

---

### Property 17: Rate Limit Reset Invariant

*For any* user U and rate limit window W of duration D seconds, after the window expires (current time > W.start + D), the request counter for U resets to 0 and the next request from U is processed normally without a 429 response.

**Validates: Requirements 26.5**

---

### Property 18: Free Plan Meeting Count Enforcement

*For any* Free_Plan user U: if U has recorded exactly 5 or more meetings in the current calendar month, `start_meeting(U)` returns HTTP 403; if U has recorded fewer than 5 meetings, `start_meeting(U)` returns HTTP 200 with a valid `meeting_id`.

**Validates: Requirements 21.1, 5.6**

---

### Property 19: Audio Deletion Invariant

*For any* completed meeting M, a query to Cloudflare R2 for objects with key prefix `audio/{M.id}/` returns an empty result set — no audio data persists after transcription completes.

**Validates: Requirements 24.6**

---

### Property 20: Reconnection Completeness Invariant

*For any* meeting M where one or more WebSocket disconnections occurred during recording, the `full_text` stored in the transcripts table for M equals the transcript that would have been produced had no disconnection occurred — the final content is identical regardless of network interruptions.

**Validates: Requirements 25.6, 6.6**

---

### Property 21: No Duplicate Chunk Processing

*For any* chunk C transmitted during a reconnection scenario (potentially buffered and retransmitted), C appears exactly once in the final transcript for the meeting — the deduplication mechanism prevents any chunk from being processed more than once.

**Validates: Requirements 25.7**

---

## Error Handling

### Error Response Format

All API errors return a consistent JSON envelope:
```json
{
  "error": "PLAN_LIMIT_EXCEEDED",
  "message": "You have reached the 5 meeting limit for the Free plan this month.",
  "request_id": "req_abc123"
}
```

### Error Matrix by Layer

| Layer | Error Type | Handling Strategy |
|-------|-----------|-------------------|
| Pydantic validation | 422 Unprocessable Entity | Auto-generated by FastAPI with field-level detail |
| Auth | 401 Unauthorized | Generic message (no user enumeration) |
| Plan enforcement | 403 Forbidden | Clear upgrade prompt in message |
| Not found | 404 Not Found | Resource type in message |
| Rate limit | 429 Too Many Requests | `Retry-After` header set |
| LLM failure | Celery retry → meeting "failed" | Push notification + retry button in app |
| Whisper chunk failure | Log + skip chunk | Continue processing subsequent chunks |
| WebSocket disconnect | Client-side buffer + backoff | Transparent to user |
| Embedding API failure | Retry 3x with backoff | Log failure, continue with remaining chunks |
| R2 upload failure | Log + continue | Audio not stored (privacy-safe failure) |
| DB connection failure | 503 Service Unavailable | /health endpoint reflects DB status |

### Structured Logging

Every request logged as JSON:
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "request_id": "req_abc123",
  "user_id": "uuid-or-null",
  "method": "POST",
  "endpoint": "/meetings/start",
  "status_code": 200,
  "response_time_ms": 45,
  "level": "INFO"
}
```

---

## Testing Strategy

### Dual Testing Approach

Both unit tests and property-based tests are required. Unit tests verify specific examples, edge cases, and integration points. Property-based tests verify universal correctness across all inputs. Together they provide comprehensive coverage.

### Property-Based Testing

**Framework:** Hypothesis (Python backend) + fast-check (TypeScript frontend)

**Configuration:** Minimum 100 iterations per property test (due to randomization).

**Tag format:** `# Feature: meet-mind, Property N: <property_text>`

Each correctness property from the design document is implemented by exactly one property-based test:

| Property | Test Location | Hypothesis Strategy |
|----------|--------------|---------------------|
| P1: Registration Round-Trip | `tests/test_auth_properties.py` | `st.text()` for name, `st.emails()` for email, `st.text(min_size=8)` for password |
| P2: Login Round-Trip | `tests/test_auth_properties.py` | Generate registered users, run full token cycle |
| P3: Meeting ID Uniqueness | `tests/test_meeting_properties.py` | `st.integers(min_value=2, max_value=50)` for N, generate N start requests |
| P4: Transcript Completeness | `tests/test_transcription_properties.py` | `st.lists(st.binary(min_size=1))` for chunk sequences |
| P5: Summary JSON Structure | `tests/test_summarization_properties.py` | `st.text(min_size=10)` for transcripts |
| P6: Task Grounding | `tests/test_summarization_properties.py` | Generate transcripts with known names, verify assignees |
| P7: Search Subset | `tests/test_meeting_properties.py` | `st.text()` for queries, `st.lists(meeting_strategy)` for meetings |
| P8: Date Filter Correctness | `tests/test_meeting_properties.py` | `st.dates()` for ranges, verify all results in range |
| P9: Deletion Cascade | `tests/test_meeting_properties.py` | Create full meeting with all related data, delete, verify |
| P10: Kanban Partition | `tests/test_task_properties.py` | `st.lists(task_strategy)` with random statuses |
| P11: Task Status Idempotence | `tests/test_task_properties.py` | `st.sampled_from(['todo','in_progress','done'])` |
| P12: Mood Score Range | `tests/test_mood_properties.py` | `st.integers(min_value=1, max_value=5)` for valid, `st.integers().filter(lambda x: not 1<=x<=5)` for invalid |
| P13: Analytics Count | `tests/test_analytics_properties.py` | Generate meetings with known statuses, verify count |
| P14: Analytics Hours | `tests/test_analytics_properties.py` | `st.integers(min_value=0)` for duration_seconds, verify calculation |
| P15: Embedding Dimension | `tests/test_embedding_properties.py` | `st.text(min_size=1)` for any input text |
| P16: Embedding Round-Trip | `tests/test_embedding_properties.py` | Generate chunks, embed, store, query, verify top result |
| P17: Rate Limit Reset | `tests/test_rate_limit_properties.py` | Simulate window expiry, verify counter reset |
| P18: Free Plan Enforcement | `tests/test_plan_properties.py` | `st.integers(min_value=0, max_value=10)` for meeting count |
| P19: Audio Deletion | `tests/test_privacy_properties.py` | Complete meeting flow, verify R2 cleanup |
| P20: Reconnection Completeness | `tests/test_websocket_properties.py` | Simulate random disconnection points in chunk sequence |
| P21: No Duplicate Processing | `tests/test_websocket_properties.py` | Simulate retransmission, verify deduplication |

### Unit Testing

**Framework:** Pytest + httpx (async) for backend; Jest + React Native Testing Library for frontend.

**Coverage target:** ≥ 80% line coverage across all service modules (NFR-4.1).

Key unit test areas:
- Auth: register/login/refresh happy paths, duplicate email, wrong password, expired token
- Meeting: start/stop/list/delete, plan limit boundary conditions (exactly 5 meetings)
- Transcription: Whisper mock, chunk sequencing, failed chunk skip
- Summarization: valid JSON response, malformed response retry, final failure → "failed" status
- Embedding: chunking algorithm (token counts, overlap), dimension assertion
- RAG: no-results path, source citation in response
- Task: status update, Kanban grouping, idempotent update
- Mood: score validation (1-5 boundary), upsert behaviour
- Analytics: count and hours calculation with known data
- Plan enforcement: each plan tier, each restricted feature
- Rate limiting: sliding window counter, 429 response, Retry-After header
- WebSocket: connection lifecycle, reconnection backoff sequence

### Integration Tests

- Full meeting lifecycle: start → stream chunks → stop → summarize → retrieve summary
- Auth flow: register → login → refresh → logout
- RAG pipeline: embed transcript → ask question → verify answer contains source
- Deletion cascade: create meeting with all data → delete → verify all tables empty
- Plan enforcement: Free user hitting each limit (meetings, duration, features)

### Frontend Tests (Jest + RNTL)

- Zustand store actions: login, logout, appendTranscript, bufferChunk
- React Query hooks: useQuery refetch interval during "processing" status
- Axios interceptor: 401 → refresh → retry
- WebSocket reconnection logic: backoff timing, buffer flush order
- Screen rendering: Home Dashboard empty state, Meeting Summary loading state

### Property Test Example (Hypothesis)

```python
# Feature: meet-mind, Property 10: Kanban Partition Invariant
@given(
    tasks=st.lists(
        st.builds(Task, status=st.sampled_from(['todo', 'in_progress', 'done'])),
        min_size=0, max_size=50
    )
)
@settings(max_examples=100)
async def test_kanban_partition_invariant(tasks, db_session, test_user):
    # Insert tasks for test_user
    for task in tasks:
        task.user_id = test_user.id
        db_session.add(task)
    await db_session.commit()

    todo = await task_service.list_tasks(test_user.id, status='todo')
    in_progress = await task_service.list_tasks(test_user.id, status='in_progress')
    done = await task_service.list_tasks(test_user.id, status='done')
    all_tasks = await task_service.list_tasks(test_user.id)

    assert len(todo) + len(in_progress) + len(done) == len(all_tasks)
```

### Property Test Example (fast-check, Frontend)

```typescript
// Feature: meet-mind, Property 11: Task Status Update Idempotence
test('task status update is idempotent', async () => {
  await fc.assert(
    fc.asyncProperty(
      fc.constantFrom('todo', 'in_progress', 'done'),
      async (status) => {
        const task = await createTestTask({ status });
        await api.updateTaskStatus(task.id, status);
        const result = await api.getTask(task.id);
        expect(result.status).toBe(status);
      }
    ),
    { numRuns: 100 }
  );
});
```
