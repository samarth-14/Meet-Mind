# 🧠 MeetMind
## AI-Powered Meeting Copilot — Mobile Application
**Product Requirements Document · v1.0 · 2025**

| Stack | Platform | AI Core | Status |
|-------|----------|---------|--------|
| React Native + FastAPI | iOS & Android | Whisper + GPT-4o | Pre-development |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Market Research](#3-market-research)
4. [Competitor Analysis](#4-competitor-analysis)
5. [Unique Selling Propositions](#5-unique-selling-propositions-usps)
6. [Features](#6-features)
7. [User Personas](#7-user-personas)
8. [Technology Stack](#8-technology-stack)
9. [System Architecture](#9-system-architecture)
10. [API Endpoints Reference](#10-api-endpoints-reference)
11. [Mobile Screens](#11-mobile-screens)
12. [MVP Development Roadmap](#12-mvp-development-roadmap)
13. [Risk Analysis](#13-risk-analysis)
14. [Monetisation Strategy](#14-monetisation-strategy)
15. [Success Metrics](#15-success-metrics)

---

## 1. Executive Summary

MeetMind is a mobile-first AI meeting intelligence platform built for small teams, freelancers, and B2B businesses. It listens to meetings in real time, transcribes speech using Whisper, and uses large language models to produce structured summaries, extract action items, track team mood, and answer natural language questions about past meetings.

The core problem MeetMind solves is simple: people waste enormous time in meetings and forget most of what was discussed within 24 hours. Existing tools either require desktop access, expensive subscriptions, or work only with specific platforms like Zoom or Google Meet. MeetMind works with any meeting, on any platform, directly from your phone.

| | |
|---|---|
| **Mission** | Make every meeting count by turning spoken conversations into structured, searchable, actionable intelligence. |
| **Target Users** | Small business owners, freelancers, startup teams, project managers, and students — anyone who attends more than 3 meetings per week. |
| **Revenue Model** | Freemium — free tier for up to 5 meetings/month. Pro at ₹499/month for unlimited meetings. Team plan at ₹1999/month for up to 10 users. |

---

## 2. Problem Statement

### 2.1 The Core Problem

Research by Harvard Business Review shows that professionals spend an average of 23 hours per week in meetings — up from under 10 hours in the 1960s. Despite this, 71% of senior managers say most meetings are unproductive, and studies show that 90% of attendees admit to daydreaming during meetings while 73% do other work.

The real damage happens after the meeting ends. Without a clear record, action items get forgotten, decisions get revisited unnecessarily, and team members misremember what was agreed. This costs businesses an estimated $37 billion per year in lost productivity in the US alone.

### 2.2 Specific Pain Points

- Manual note-taking is distracting — you cannot fully participate while writing notes
- Meeting summaries written by humans are inconsistent, biased, and time-consuming
- Action items fall through the cracks when buried inside long email threads or chat messages
- There is no easy way to search past meetings — "what did we decide about X last month?" is impossible to answer quickly
- Meeting fatigue and team burnout is invisible to managers until it becomes a resignation
- Most AI meeting tools are desktop-only, tied to specific video platforms, or priced for enterprises

### 2.3 Why Mobile-First?

A significant portion of meetings — especially client calls, field team check-ins, and casual stand-ups — happen over the phone, not on desktop video calls. No major AI meeting tool targets this use case. MeetMind's React Native app fills this gap entirely.

---

## 3. Market Research

### 3.1 Market Size

| Segment | Size | Description |
|---------|------|-------------|
| Total Addressable Market (TAM) | $4.7 Billion | Global meeting productivity software market (2024) |
| Serviceable Addressable Market (SAM) | $820 Million | AI-powered transcription & summarization tools |
| Serviceable Obtainable Market (SOM) | $12 Million | Mobile-first meeting tools in India + SE Asia |

### 3.2 Market Trends

- The AI productivity tools market is growing at 24.3% CAGR (2024–2029)
- Remote and hybrid work has normalized meeting culture globally, increasing demand for meeting intelligence
- India's SaaS market is growing rapidly — Indian SMBs are increasingly adopting productivity software
- Voice AI accuracy (Whisper, Deepgram) has crossed the threshold for reliable commercial use
- Demand for privacy-first tools is rising — users want on-device or self-hosted AI options

---

## 4. Competitor Analysis

### 4.1 Direct Competitors

| Product | Platform | Price/mo | Mobile App | Offline | Key Weakness |
|---------|----------|----------|------------|---------|--------------|
| Otter.ai | Web + Mobile | $16.99 | Yes (limited) | No | Expensive, US-centric pricing |
| Fireflies.ai | Web only | $18 | No | No | No mobile recording, bot-join only |
| Notion AI | Web + Mobile | $10 addon | Partial | No | Not meeting-specific, no transcription |
| Fathom | Desktop | Free / $19 | No | No | Zoom-only, no mobile support |
| Tactiq | Chrome ext | $12 | No | No | Browser extension only |
| **MeetMind** | **Mobile-first** | **Free / ₹499** | **Yes (core)** | **Yes** | **— (our product)** |

### 4.2 Competitive Advantages of MeetMind

- Only mobile-native product — works for phone calls, not just video meetings
- Works completely offline for transcription (Whisper base model on device)
- India-first pricing — ₹499/month vs $16–19/month for US competitors
- RAG-powered AI chat — search and query past meetings conversationally
- Team mood tracking — no competitor offers this as an integrated feature
- Open-source AI stack — no dependency on expensive proprietary transcription APIs

---

## 5. Unique Selling Propositions (USPs)

### USP 1 — Works on any call, on any platform

- Competitors like Fireflies and Fathom work only when a bot joins a Zoom or Google Meet call
- MeetMind uses the phone's microphone directly — it records any meeting, phone call, in-person discussion, or voice note
- No integrations required to start using it

### USP 2 — Ask your meetings questions

- MeetMind builds a searchable knowledge base of everything ever discussed in your meetings
- Using RAG (Retrieval-Augmented Generation), users can ask questions like "What was the budget approved for the Q3 campaign?"
- No competitor offers this as a mobile feature

### USP 3 — Team mood & burnout intelligence

- After each meeting, a 5-second mood check-in captures how participants felt
- Over time, the AI identifies patterns — which meeting types drain energy, which team members show burnout signals
- Managers get a weekly team health report they cannot get anywhere else

### USP 4 — India-first pricing & offline capability

- Priced at ₹499/month — accessible for Indian freelancers and small businesses
- Whisper runs locally on device, meaning transcription works without internet
- No US dollar pricing, no credit card required for the free tier

---

## 6. Features

### 6.1 Core Feature — Live Meeting Intelligence

The primary capability of MeetMind. When a user taps "Start Meeting", the app begins recording audio through the device microphone. Audio is streamed in 30-second chunks to the FastAPI backend over WebSocket. Each chunk is processed by Whisper and the partial transcript is immediately sent back to the app, creating a live scrolling transcription experience.

When recording is stopped, the complete transcript is sent to an LLM (GPT-4o or Claude) with a structured prompt. The model returns a JSON object containing the meeting summary, list of decisions made, and action items with assignee names detected from the conversation. This entire pipeline completes in under 15 seconds for a 30-minute meeting.

### 6.2 Feature List

| Feature | Priority | Description |
|---------|----------|-------------|
| Live transcription | Core | Real-time speech-to-text via Whisper streamed over WebSocket |
| AI meeting summary | Core | Auto-generated paragraph summary after every meeting ends |
| Action item extraction | Core | AI detects tasks and assignee names from conversation |
| Decision log | Core | AI identifies and lists every decision made in the meeting |
| Task board | Side | Kanban board aggregating all tasks from all meetings |
| Meeting history | Side | Searchable archive of all past meetings with transcripts |
| Analytics dashboard | Side | Charts for meetings/week, sentiment trends, task completion |
| Mood check-in | Side | 5-emoji post-meeting mood capture with heatmap over time |
| Pre-meeting brief | Side | Push notification 15 min before meetings with agenda and pending tasks |
| Team mood heatmap | Side | Manager view showing team stress levels over weeks |
| AI chat — Ask a meeting | Bonus | RAG-based chat to query any past meeting in natural language |
| Integrations | Bonus | Export summaries to Slack, Notion, Trello, or email |
| Speaker diarization | Future | Identify who said what using voice fingerprinting |
| Sentiment per speaker | Future | Track individual participant sentiment across meetings |

---

## 7. User Personas

### Persona A — The Founder

| Field | Detail |
|-------|--------|
| Name | Arjun, 34 — Startup Founder |
| Situation | Runs a 12-person team, attends 8–10 meetings per day across phone calls and Google Meet |
| Pain | Forgets follow-ups, cannot remember who committed to what, team feels burnt out |
| Uses MeetMind for | Post-meeting summaries, task delegation tracking, weekly mood reports for his team |
| Willing to pay | Yes — ₹1999/month for the Team plan |

### Persona B — The Freelancer

| Field | Detail |
|-------|--------|
| Name | Priya, 27 — Freelance Consultant |
| Situation | Works with 6 clients simultaneously, all communication over WhatsApp calls and Zoom |
| Pain | Clients dispute what was agreed, no documentation, manual notes take too long |
| Uses MeetMind for | Recording client calls, sharing auto-generated summaries as meeting minutes |
| Willing to pay | Yes — ₹499/month (Pro plan, writes off as business expense) |

### Persona C — The Student

| Field | Detail |
|-------|--------|
| Name | Rahul, 21 — CS Student / Intern |
| Situation | Part of multiple college project groups and internship stand-ups |
| Pain | Misses context from meetings, has to message teammates constantly to catch up |
| Uses MeetMind for | Free tier — 5 meetings/month, occasional summaries for project reviews |
| Willing to pay | Not immediately, but will upgrade when he joins a company |

---

## 8. Technology Stack

### 8.1 Frontend — React Native

The mobile application is built using React Native with Expo. This choice enables a single codebase to deploy on both iOS and Android, while Expo's managed workflow handles complex native modules like audio recording, push notifications, and device permissions without requiring native Xcode or Android Studio builds.

| Library | Purpose | Why chosen |
|---------|---------|------------|
| React Native 0.73+ | Core framework | Cross-platform mobile development with a single JS codebase |
| Expo SDK 50+ | Dev toolchain | Managed workflow, OTA updates, simplified native module access |
| Expo Audio | Audio capture | Mic recording, AAC format, chunked streaming |
| Expo Notifications | Push alerts | Scheduled pre-meeting briefs, task reminders |
| React Navigation v6 | Navigation | Stack + Bottom Tab Navigator for screen routing |
| Zustand | State management | Lightweight global state — auth, meetings, tasks |
| React Query (TanStack) | Data fetching | Caching, background sync, optimistic updates for API calls |
| Victory Native | Charts | Bar, line, and pie charts for analytics screen |
| Expo SecureStore | Auth storage | Encrypted JWT token storage on device |
| Axios | HTTP client | API calls with JWT interceptor auto-attached |
| React Native Reanimated | Animations | Smooth transitions, recording pulse animation |

### 8.2 Backend — FastAPI

The backend is built with FastAPI (Python). FastAPI is chosen for its native async support, which is essential for handling WebSocket audio streams without blocking, and for its automatic OpenAPI documentation generation. The Python ecosystem also provides direct access to Whisper, LangChain, and all ML libraries needed without any bridging.

| Library | Purpose | Why chosen |
|---------|---------|------------|
| FastAPI | Web framework | Async-first, automatic docs, WebSocket support built in |
| Uvicorn | ASGI server | High-performance async server for FastAPI |
| SQLAlchemy 2.0 | ORM | Async-compatible ORM for PostgreSQL queries |
| Alembic | DB migrations | Version-controlled schema migrations |
| Pydantic v2 | Validation | Request/response schema validation and serialization |
| Python-jose | Auth | JWT creation, signing, and verification |
| Passlib + bcrypt | Password hashing | Secure password storage |
| Celery + Redis | Task queue | Background jobs for LLM summarization after meeting ends |
| Python-dotenv | Config | Environment variable management |
| Pytest + httpx | Testing | Async API testing |

### 8.3 AI & Machine Learning

| Tool | Role | Notes |
|------|------|-------|
| OpenAI Whisper | Speech-to-text | Open-source, runs locally (base model free), high accuracy |
| GPT-4o / Claude 3.5 | Summarization & LLM | Structured JSON output for summaries, tasks, decisions |
| text-embedding-3-small | Embeddings | Converts transcript chunks to vectors for semantic search |
| pgvector | Vector store | PostgreSQL extension for storing and querying embeddings |
| LangChain (optional) | RAG orchestration | Simplifies the retrieval + generation pipeline for AI chat |

### 8.4 Database & Infrastructure

| Technology | Role | Notes |
|------------|------|-------|
| PostgreSQL 16 | Primary database | Relational store for users, meetings, transcripts, tasks |
| pgvector extension | Vector DB | Native vector similarity search without a separate service |
| Redis 7 | Cache + queue | Celery broker, session cache, rate limiting |
| Docker + Docker Compose | Containerization | Consistent dev and prod environments |
| Railway / Render | Deployment | Free tier deployment for MVP phase |
| Cloudflare R2 | File storage | Audio file storage (cheaper than AWS S3) |
| GitHub Actions | CI/CD | Automated testing and deployment on push |

---

## 9. System Architecture

### 9.1 High-Level Architecture

MeetMind follows a clean client-server architecture with an async message queue for heavy AI processing. The React Native app communicates with FastAPI over two channels: a REST API for standard CRUD operations, and a WebSocket connection for real-time audio streaming during live meetings.

- **React Native App (iOS / Android)**
  - Communicates via REST (Axios) for auth, meeting history, tasks, analytics
  - Communicates via WebSocket for live audio streaming and real-time transcript delivery

- **FastAPI Backend**
  - REST endpoints handle all CRUD, auth, and analytics aggregation
  - WebSocket endpoint receives audio chunks, processes with Whisper, returns text
  - On meeting end, a Celery task is triggered asynchronously for LLM summarization

- **AI Pipeline**
  - Whisper processes each audio chunk and returns transcript text in under 2 seconds
  - On stop: full transcript sent to GPT-4o with structured prompt — returns JSON
  - Embeddings generated per transcript chunk and stored in pgvector for RAG
  - AI chat queries pgvector for relevant chunks, feeds to LLM, streams answer back

### 9.2 Database Schema — Key Tables

| Table | Key Columns | Purpose |
|-------|-------------|---------|
| users | id, name, email, password_hash, plan, created_at | Stores account info and subscription tier |
| meetings | id, user_id, title, status, duration_seconds, created_at | One row per meeting session |
| transcripts | id, meeting_id, full_text, chunks_json, created_at | Full text + chunked text for RAG |
| summaries | id, meeting_id, summary, decisions_json, tasks_json | AI output for each meeting |
| tasks | id, meeting_id, user_id, text, assignee, status, due_date | Extracted action items |
| moods | id, meeting_id, user_id, score (1-5), created_at | Post-meeting mood check-ins |
| embeddings | id, transcript_id, chunk_text, vector (1536-dim) | Stored in pgvector for semantic search |

---

## 10. API Endpoints Reference

| Endpoint | Access | Description |
|----------|--------|-------------|
| `POST /auth/register` | Public | Register with name, email, password — returns JWT |
| `POST /auth/login` | Public | Login — returns access + refresh tokens |
| `POST /auth/google` | Public | Google OAuth flow — returns JWT |
| `POST /meetings/start` | Auth | Creates meeting record, returns meeting_id |
| `POST /meetings/{id}/stop` | Auth | Triggers Celery job for AI summarization |
| `GET /meetings` | Auth | List all meetings, supports ?search= and ?date= |
| `GET /meetings/{id}` | Auth | Full meeting detail — transcript, summary, tasks |
| `DELETE /meetings/{id}` | Auth | Delete meeting and all related data |
| `WS /ws/transcribe/{id}` | Auth | WebSocket — stream audio chunks, receive transcript text |
| `GET /tasks` | Auth | All tasks — supports ?status=todo\|in_progress\|done |
| `POST /tasks/{id}/status` | Auth | Update task status |
| `POST /mood` | Auth | Save post-meeting mood score (1-5) |
| `GET /mood/heatmap` | Auth | Weekly mood data for heatmap visualization |
| `GET /analytics/summary` | Auth | Aggregated weekly stats — meetings, hours, sentiment |
| `POST /ai/ask` | Auth | RAG chat — question → vector search → LLM → streamed answer |

---

## 11. Mobile Screens

| Screen | Phase | Description |
|--------|-------|-------------|
| S01 — Onboarding | Phase 1 | Email/password + Google sign-in. JWT saved to SecureStore on success |
| S02 — Home dashboard | Phase 1 | Today's stats, recent meetings list, Start Meeting FAB button |
| S03 — Live recording | Phase 2 | Pulsing indicator, live transcript scroll, Stop & Summarize button |
| S04 — Meeting summary | Phase 2 | AI summary, decisions, action items — auto-opens after recording |
| S05 — Meeting history | Phase 3 | Searchable list of all past meetings with date filter |
| S06 — Task board | Phase 3 | Kanban — Todo / In Progress / Done columns with drag-and-drop |
| S07 — Analytics | Phase 3 | Bar chart (meetings/week), line (sentiment), pie (task status) |
| S08 — Team mood | Phase 3 | Mood check-in sheet + weekly heatmap for managers |
| S09 — AI chat | Phase 4 | Chat UI for querying past meetings — SSE-streamed responses |
| S10 — Integrations | Phase 4 | Connect Slack, Notion, Trello — one-tap summary export |
| S11 — Settings | Phase 1 | Profile, notifications, plan, team invite link |

---

## 12. MVP Development Roadmap

| Phase | Timeline | Deliverables |
|-------|----------|--------------|
| Phase 1 — Foundation | Week 1 | FastAPI project setup, PostgreSQL schema, JWT auth, React Native init, tab navigator, Axios client |
| Phase 2 — Core feature | Week 2–3 | Expo Audio, WebSocket stream, Whisper integration, LLM summarization pipeline, Meeting summary screen |
| Phase 3 — Side features | Week 4–5 | Task board, meeting history, mood check-in, analytics charts, push notifications |
| Phase 4 — AI chat + integrations | Week 6 | pgvector setup, embeddings pipeline, RAG endpoint, AI chat screen, Slack/Notion export |
| Phase 5 — Polish & deploy | Week 7 | Error handling, loading states, offline support, Docker deploy to Railway/Render, TestFlight beta |

---

## 13. Risk Analysis

| Risk | Level | Mitigation |
|------|-------|------------|
| Whisper latency | Medium | On-device Whisper (base model) may lag on low-end phones. Use cloud Whisper API as fallback, cache results |
| LLM API cost | High | GPT-4o at $5/1M tokens — high volume could be expensive. Use GPT-4o-mini for non-critical tasks, cache summaries |
| Audio permission denial | Low | Users may deny mic access. Clear onboarding explaining why mic is needed, graceful fallback |
| WebSocket disconnects | Medium | Poor network can drop WS connection mid-meeting. Auto-reconnect logic, buffer chunks locally |
| Data privacy | High | Meeting audio is sensitive. On-device Whisper option, clear privacy policy, no audio stored after transcription |

---

## 14. Monetisation Strategy

### 14.1 Pricing Tiers

| Plan | Price | Includes |
|------|-------|---------|
| Free | ₹0 / month | 5 meetings/month, 30 min max per meeting, basic summary only, 7-day history |
| Pro | ₹499 / month | Unlimited meetings, unlimited duration, all features, 1-year history, priority support |
| Team | ₹1999 / month | Up to 10 users, shared task board, team mood dashboard, admin controls, API access |

### 14.2 Growth Strategy

- Launch free tier first — acquire users with zero friction
- Referral program — invite 3 colleagues, get 1 month Pro free
- Target freelancer communities on LinkedIn and Twitter India
- Product Hunt launch for global exposure
- B2B outreach to startup founders — team plan upsell

---

## 15. Success Metrics

| Metric | Target & Notes |
|--------|---------------|
| DAU / MAU Ratio | Target > 40% — indicates daily habit formation |
| Meetings recorded per user/week | Target > 3 — core engagement signal |
| AI summary satisfaction | Target > 4.2 / 5 — in-app rating after each summary |
| Free to Pro conversion | Target > 8% within 60 days of signup |
| Task completion rate | % of AI-extracted tasks marked done within 7 days |
| Churn rate | Target < 5% monthly for Pro users |
| App Store rating | Target > 4.5 stars within first 100 reviews |

---

## Document Information

| | |
|---|---|
| **Version** | 1.0 — Initial draft |
| **Tech Stack** | React Native (Expo) + FastAPI (Python) + PostgreSQL + Redis + Whisper + GPT-4o |
| **Author note** | This document covers the complete product scope for MeetMind MVP. Build phases are designed for a solo developer working part-time over 7 weeks. |

---

*MeetMind — Product Requirements Document · Confidential*
