# Requirements Document

## Introduction

MeetMind is a mobile-first AI meeting intelligence platform for small teams, freelancers, and students.
The application records meetings via the device microphone, streams audio to a backend for real-time
transcription using OpenAI Whisper, and uses large language models (GPT-4o / Claude 3.5) to produce
structured summaries, extract action items, log decisions, track team mood, and answer natural language
questions about past meetings via a RAG-powered chat interface.

The system operates on a freemium model: Free (5 meetings/month), Pro (₹499/month, unlimited), and
Team (₹1999/month, up to 10 users with shared features). It targets iOS and Android via a React Native
(Expo) frontend backed by a FastAPI (Python) server, PostgreSQL 16, Redis 7, and Cloudflare R2.

---

## Glossary

- **App**: The MeetMind React Native mobile application running on iOS or Android.
- **Backend**: The FastAPI Python server that handles REST, WebSocket, and AI pipeline operations.
- **Auth_Service**: The component responsible for user registration, login, JWT issuance, and token refresh.
- **Meeting_Service**: The component that manages meeting lifecycle — start, stop, list, retrieve, delete.
- **Transcription_Service**: The Whisper-based component that converts audio chunks to text in real time.
- **Summarization_Service**: The LLM-based component (GPT-4o / Claude 3.5) that produces structured meeting output.
- **Task_Service**: The component that manages extracted action items and their Kanban status.
- **Mood_Service**: The component that records post-meeting mood scores and aggregates heatmap data.
- **Analytics_Service**: The component that aggregates weekly meeting statistics and sentiment trends.
- **Embedding_Service**: The component that converts transcript chunks to 1536-dimensional vectors for RAG.
- **RAG_Service**: The Retrieval-Augmented Generation component that answers natural language queries about meetings.
- **Notification_Service**: The component that schedules and delivers push notifications for pre-meeting briefs.
- **Integration_Service**: The component that exports meeting summaries to Slack, Notion, Trello, or email.
- **Plan_Enforcer**: The component that validates subscription tier limits before allowing operations.
- **User**: A registered account holder interacting with the App.
- **Meeting**: A recorded audio session with associated transcript, summary, decisions, and tasks.
- **Transcript**: The full text output of Whisper processing for a given Meeting.
- **Summary**: The LLM-generated paragraph describing the key points of a Meeting.
- **Decision**: An item identified by the LLM as a decision made during a Meeting.
- **Task**: An action item extracted by the LLM from a Meeting, with optional assignee and due date.
- **Mood_Score**: An integer in the range [1, 5] representing a User's post-meeting emotional state.
- **Embedding**: A 1536-dimensional float vector representing a transcript chunk for semantic search.
- **JWT**: A JSON Web Token used for authenticating API requests.
- **Refresh_Token**: A long-lived token used to obtain a new JWT without re-entering credentials.
- **Free_Plan**: The ₹0/month tier — 5 meetings/month, 30-minute max per meeting, 7-day history.
- **Pro_Plan**: The ₹499/month tier — unlimited meetings, all features, 1-year history.
- **Team_Plan**: The ₹1999/month tier — up to 10 users, shared task board, team mood dashboard, admin controls.
- **Chunk**: A 30-second segment of audio sent over WebSocket for incremental transcription.
- **RAG**: Retrieval-Augmented Generation — a technique combining vector search with LLM generation.
- **Kanban**: A board with columns Todo, In_Progress, and Done for visualising task status.
- **Heatmap**: A calendar-style visualisation of Mood_Score values aggregated by week.
- **SecureStore**: Expo's encrypted on-device key-value store used to persist JWTs.
- **Celery**: The distributed task queue used to run LLM summarization asynchronously after a Meeting ends.
- **pgvector**: The PostgreSQL extension used to store and query Embeddings via cosine similarity.

---

## Requirements


### Requirement 1: User Registration

**User Story:** As a new user, I want to create an account with my email and password, so that I can access MeetMind and have my meetings and data persisted securely.

#### Acceptance Criteria

1. WHEN a registration request is received with a valid name, email address, and password of at least 8 characters, THE Auth_Service SHALL create a new user record and return a signed JWT.
2. WHEN a registration request is received with an email address that already exists in the system, THE Auth_Service SHALL return an HTTP 409 Conflict error with a descriptive message.
3. WHEN a registration request is received with a password shorter than 8 characters, THE Auth_Service SHALL return an HTTP 422 Unprocessable Entity error identifying the invalid field.
4. WHEN a registration request is received with a malformed email address, THE Auth_Service SHALL return an HTTP 422 Unprocessable Entity error identifying the invalid field.
5. THE Auth_Service SHALL store passwords as bcrypt hashes and SHALL NOT store plaintext passwords.
6. WHEN a new user is created, THE Auth_Service SHALL assign the Free_Plan to that user by default.
7. FOR ALL valid registration inputs (name, email, password), THE Auth_Service SHALL produce a JWT that is accepted by subsequent authenticated endpoints without error (round-trip property).

---

### Requirement 2: User Login

**User Story:** As a registered user, I want to log in with my email and password, so that I can access my meetings and data on any device.

#### Acceptance Criteria

1. WHEN a login request is received with a valid email and correct password, THE Auth_Service SHALL return both an access JWT and a Refresh_Token.
2. WHEN a login request is received with a valid email and incorrect password, THE Auth_Service SHALL return an HTTP 401 Unauthorized error.
3. WHEN a login request is received with an email that does not exist, THE Auth_Service SHALL return an HTTP 401 Unauthorized error without revealing whether the email exists.
4. THE Auth_Service SHALL issue access JWTs with an expiry of no more than 60 minutes.
5. THE Auth_Service SHALL issue Refresh_Tokens with an expiry of no more than 30 days.
6. WHEN a valid Refresh_Token is submitted to the token refresh endpoint, THE Auth_Service SHALL return a new access JWT valid for the standard access token lifetime.
7. WHEN an expired or invalid Refresh_Token is submitted, THE Auth_Service SHALL return an HTTP 401 Unauthorized error.
8. FOR ALL valid login sequences (login → use access token → refresh → use new access token), THE Auth_Service SHALL maintain an authenticated session without requiring re-entry of credentials (round-trip property).

---

### Requirement 3: Google OAuth Sign-In

**User Story:** As a user, I want to sign in with my Google account, so that I can access MeetMind without creating a separate password.

#### Acceptance Criteria

1. WHEN a valid Google OAuth authorization code is submitted to the Google OAuth endpoint, THE Auth_Service SHALL exchange it for a Google identity token, create or retrieve the corresponding user record, and return a signed JWT.
2. WHEN an invalid or expired Google OAuth code is submitted, THE Auth_Service SHALL return an HTTP 401 Unauthorized error.
3. WHEN a Google OAuth sign-in is performed for an email that already exists as a password-registered account, THE Auth_Service SHALL link the Google identity to the existing account and return a valid JWT.
4. THE Auth_Service SHALL NOT require a password for accounts created via Google OAuth.

---

### Requirement 4: JWT Storage and Session Persistence

**User Story:** As a user, I want my login session to persist across app restarts, so that I do not have to log in every time I open the app.

#### Acceptance Criteria

1. WHEN a JWT is successfully obtained, THE App SHALL store it in SecureStore using encrypted key-value storage.
2. WHEN the App is launched and a valid JWT exists in SecureStore, THE App SHALL restore the authenticated session without prompting the user to log in.
3. WHEN the stored JWT is expired, THE App SHALL automatically use the stored Refresh_Token to obtain a new JWT before making API requests.
4. IF the Refresh_Token is also expired or absent, THEN THE App SHALL redirect the user to the login screen.
5. THE App SHALL attach the JWT as a Bearer token in the Authorization header of every authenticated API request via the Axios interceptor.

---

### Requirement 5: Meeting Start

**User Story:** As a user, I want to start a new meeting recording with a single tap, so that I can capture the conversation without manual setup.

#### Acceptance Criteria

1. WHEN the user taps the Start Meeting button, THE Meeting_Service SHALL create a new Meeting record with status "recording" and return a unique meeting_id.
2. WHEN a Meeting record is created, THE App SHALL immediately open the Live Recording screen (S03) and establish a WebSocket connection to the transcription endpoint for that meeting_id.
3. THE App SHALL request microphone permission before starting a recording; IF permission is denied, THEN THE App SHALL display an explanatory message and SHALL NOT attempt to record.
4. WHILE a meeting is in "recording" status, THE App SHALL display a pulsing visual indicator to confirm active recording.
5. FOR ALL authenticated users, each call to the meeting start endpoint SHALL produce a meeting_id that is unique across all meetings in the system (uniqueness invariant).
6. WHERE the user is on the Free_Plan, THE Plan_Enforcer SHALL reject the meeting start request with HTTP 403 if the user has already recorded 5 or more meetings in the current calendar month.

---

### Requirement 6: Live Audio Streaming and Real-Time Transcription

**User Story:** As a user, I want to see a live scrolling transcript of the conversation as it happens, so that I can verify the recording is working and follow along.

#### Acceptance Criteria

1. WHEN a WebSocket connection is established for a meeting, THE App SHALL capture audio from the device microphone in AAC format and transmit it to the Backend in 30-second Chunks.
2. WHEN a Chunk is received by the Backend, THE Transcription_Service SHALL process it with Whisper and return the partial transcript text to the App within 2 seconds of receiving the Chunk.
3. WHILE a meeting is being recorded, THE App SHALL append each returned partial transcript to a scrolling text view, auto-scrolling to the latest content.
4. IF the WebSocket connection is dropped during a recording, THEN THE App SHALL buffer subsequent audio Chunks locally and attempt to reconnect automatically within 5 seconds.
5. WHEN the WebSocket connection is re-established after a disconnect, THE App SHALL transmit any locally buffered Chunks to the Backend in order, ensuring no audio is lost.
6. FOR ALL sequences of Chunks transmitted during a meeting, the concatenation of all partial transcripts returned by THE Transcription_Service SHALL equal the full Transcript stored for that meeting (transcript completeness invariant).
7. THE Transcription_Service SHALL process each Chunk independently such that a single failed Chunk does not prevent subsequent Chunks from being processed.
8. WHERE the device has no network connectivity, THE App SHALL use the on-device Whisper base model to transcribe audio locally and sync the Transcript to the Backend when connectivity is restored.

---

### Requirement 7: Meeting Stop and AI Summarization Pipeline

**User Story:** As a user, I want to stop the recording and automatically receive a structured AI summary, so that I have a complete record of the meeting without any manual effort.

#### Acceptance Criteria

1. WHEN the user taps the Stop & Summarize button, THE Meeting_Service SHALL update the meeting status to "processing" and enqueue a Celery summarization task.
2. WHEN the Celery task executes, THE Summarization_Service SHALL send the full Transcript to the LLM with a structured prompt requesting a JSON response containing: a "summary" string, a "decisions" array, and a "tasks" array.
3. THE Summarization_Service SHALL complete the full summarization pipeline in under 15 seconds for a Transcript of up to 30 minutes of speech.
4. WHEN the LLM response is received, THE Summarization_Service SHALL validate that the response contains all three required fields (summary, decisions, tasks) before persisting it.
5. IF the LLM response is malformed or missing required fields, THEN THE Summarization_Service SHALL retry the LLM call up to 2 additional times before marking the meeting status as "failed" and notifying the user.
6. WHEN summarization completes successfully, THE Meeting_Service SHALL update the meeting status to "done" and THE App SHALL automatically navigate to the Meeting Summary screen (S04).
7. FOR ALL valid Transcripts, THE Summarization_Service SHALL produce a summary JSON where the "tasks" array contains only items whose assignee names appear verbatim or as clear references within the Transcript text (grounding invariant).
8. THE Summarization_Service SHALL generate Embeddings for each Transcript Chunk and store them via THE Embedding_Service upon successful summarization.

---

### Requirement 8: Meeting Summary Screen

**User Story:** As a user, I want to view the AI-generated summary, decisions, and action items for a meeting on a dedicated screen, so that I can quickly review what was discussed and what needs to be done.

#### Acceptance Criteria

1. WHEN the Meeting Summary screen (S04) is opened for a completed meeting, THE App SHALL display the AI-generated Summary paragraph, the list of Decisions, and the list of Tasks with their assignees.
2. THE App SHALL display the meeting title, date, and duration at the top of the Meeting Summary screen.
3. WHEN a Task is displayed on the Meeting Summary screen, THE App SHALL show its current status (Todo, In_Progress, or Done).
4. WHEN the user taps a Task on the Meeting Summary screen, THE App SHALL allow the user to update the Task status inline.
5. IF a meeting is in "processing" status when the screen is opened, THEN THE App SHALL display a loading indicator and poll for completion at 3-second intervals.
6. IF a meeting has status "failed", THEN THE App SHALL display an error message and offer a retry option.

---

### Requirement 9: Action Item Extraction

**User Story:** As a user, I want the AI to automatically extract action items and their assignees from the meeting conversation, so that tasks are captured without manual effort.

#### Acceptance Criteria

1. WHEN the Summarization_Service processes a Transcript, THE Summarization_Service SHALL extract all action items mentioned in the conversation and populate the "tasks" array in the summary JSON.
2. WHEN an action item is extracted, THE Task_Service SHALL create a corresponding Task record with status "todo", the extracted text, and the detected assignee name (if any).
3. THE Task_Service SHALL associate each Task with both the source Meeting and the User who owns the meeting.
4. WHEN no action items are detected in a Transcript, THE Summarization_Service SHALL return an empty "tasks" array rather than null or an error.
5. FOR ALL extracted Tasks, the Task text SHALL be a non-empty string of at most 500 characters.
6. FOR ALL extracted Tasks where an assignee is detected, the assignee name SHALL be a non-empty string that appears as a reference within the source Transcript.

---

### Requirement 10: Decision Log

**User Story:** As a user, I want the AI to identify and list every decision made during the meeting, so that I have a clear record of what was agreed upon.

#### Acceptance Criteria

1. WHEN the Summarization_Service processes a Transcript, THE Summarization_Service SHALL identify all decisions made during the meeting and populate the "decisions" array in the summary JSON.
2. WHEN no decisions are detected in a Transcript, THE Summarization_Service SHALL return an empty "decisions" array rather than null or an error.
3. THE App SHALL display the decisions list on the Meeting Summary screen (S04) in the order they were identified.
4. FOR ALL meetings with a completed summary, the decisions array SHALL be a list of strings, each string being a non-empty description of a decision.

---

### Requirement 11: Meeting History

**User Story:** As a user, I want to browse and search all my past meetings, so that I can find specific conversations and review their content.

#### Acceptance Criteria

1. WHEN the Meeting History screen (S05) is opened, THE Meeting_Service SHALL return a paginated list of all meetings belonging to the authenticated User, ordered by creation date descending.
2. WHEN the user enters a search term in the search field, THE Meeting_Service SHALL return only meetings whose title or Transcript text contains the search term (case-insensitive).
3. WHEN the user applies a date filter, THE Meeting_Service SHALL return only meetings created within the specified date range.
4. WHEN the user taps a meeting in the history list, THE App SHALL navigate to the Meeting Summary screen (S04) for that meeting.
5. WHERE the user is on the Free_Plan, THE Meeting_Service SHALL return only meetings created within the last 7 days.
6. WHERE the user is on the Pro_Plan or Team_Plan, THE Meeting_Service SHALL return meetings created within the last 365 days.
7. FOR ALL search queries, the result set SHALL be a subset of the user's full meeting list (search subset invariant).
8. FOR ALL date-filtered queries, every meeting in the result set SHALL have a created_at timestamp within the specified date range (date filter correctness invariant).

---

### Requirement 12: Meeting Deletion

**User Story:** As a user, I want to delete a meeting and all its associated data, so that I can manage my storage and remove sensitive recordings.

#### Acceptance Criteria

1. WHEN the user requests deletion of a meeting, THE Meeting_Service SHALL delete the Meeting record and all associated Transcripts, Summaries, Tasks, Moods, and Embeddings in a single atomic operation.
2. WHEN a meeting is successfully deleted, THE Meeting_Service SHALL return HTTP 204 No Content.
3. WHEN a request is made to retrieve a deleted meeting, THE Meeting_Service SHALL return HTTP 404 Not Found.
4. WHEN a user attempts to delete a meeting that belongs to a different user, THE Meeting_Service SHALL return HTTP 403 Forbidden.
5. FOR ALL deleted meetings, no associated audio files SHALL remain in Cloudflare R2 storage after deletion completes (data cleanup invariant).
6. FOR ALL deleted meetings, a subsequent GET /meetings request SHALL not include the deleted meeting in the response (deletion visibility invariant).


### Requirement 13: Task Board (Kanban)

**User Story:** As a user, I want to view all my extracted tasks across all meetings in a Kanban board, so that I can track progress and manage my workload in one place.

#### Acceptance Criteria

1. WHEN the Task Board screen (S06) is opened, THE Task_Service SHALL return all Tasks belonging to the authenticated User, grouped by status: Todo, In_Progress, and Done.
2. THE App SHALL display the three Kanban columns (Todo, In_Progress, Done) with their respective Tasks as draggable cards.
3. WHEN the user drags a Task card from one column to another, THE Task_Service SHALL update the Task status to the target column's status.
4. WHEN the user applies a status filter (e.g., ?status=todo), THE Task_Service SHALL return only Tasks matching the specified status.
5. FOR ALL Task Board states, the total count of Tasks across all three columns SHALL equal the total number of Tasks for that user (partition invariant).
6. FOR ALL Task status updates, a subsequent GET /tasks request SHALL reflect the updated status (update visibility invariant).
7. WHEN a Task status is updated to its current value, THE Task_Service SHALL return HTTP 200 and leave the Task unchanged (idempotence property).
8. THE App SHALL display the source meeting title on each Task card so the user can trace the task's origin.

---

### Requirement 14: Mood Check-In

**User Story:** As a user, I want to record how I felt after a meeting using a simple emoji scale, so that I can track my meeting fatigue over time.

#### Acceptance Criteria

1. WHEN a meeting ends and the summary is displayed, THE App SHALL present a mood check-in prompt with 5 emoji options representing scores 1 through 5.
2. WHEN the user selects a mood emoji, THE Mood_Service SHALL record a Mood_Score for that meeting and user with the selected integer value.
3. WHEN a mood submission is received with a score outside the range [1, 5], THE Mood_Service SHALL return HTTP 422 Unprocessable Entity.
4. THE Mood_Service SHALL allow at most one Mood_Score per user per meeting; a second submission for the same meeting SHALL overwrite the first.
5. WHEN the user skips the mood check-in, THE App SHALL not submit any mood record and SHALL proceed normally.
6. FOR ALL stored Mood_Scores, the score value SHALL be an integer in the closed interval [1, 5] (mood range invariant).

---

### Requirement 15: Team Mood Heatmap

**User Story:** As a team manager, I want to see a weekly heatmap of my team's mood scores, so that I can identify burnout patterns and take action before people resign.

#### Acceptance Criteria

1. WHEN the Team Mood screen (S08) is opened by a user on the Team_Plan, THE Mood_Service SHALL return aggregated weekly Mood_Score data for all users in the team.
2. THE App SHALL render the mood data as a calendar heatmap where each cell represents one day and its colour intensity corresponds to the average Mood_Score for that day.
3. WHEN a user on the Free_Plan or Pro_Plan accesses the team mood heatmap endpoint, THE Backend SHALL return HTTP 403 Forbidden.
4. THE Mood_Service SHALL aggregate Mood_Scores by computing the arithmetic mean of all scores submitted on each day, rounded to two decimal places.
5. FOR ALL heatmap responses, every day in the requested date range SHALL have an entry (with null or 0 for days with no submissions) so the App can render a complete grid (completeness invariant).
6. WHERE no mood data exists for a given day, THE Mood_Service SHALL return a null value for that day's cell rather than omitting the day from the response.

---

### Requirement 16: Analytics Dashboard

**User Story:** As a user, I want to see charts showing my meeting activity, sentiment trends, and task completion rates, so that I can understand my productivity patterns.

#### Acceptance Criteria

1. WHEN the Analytics screen (S07) is opened, THE Analytics_Service SHALL return aggregated statistics for the current week: total meetings count, total meeting hours, and average sentiment score.
2. THE App SHALL display a bar chart of meetings per day for the current week using Victory Native.
3. THE App SHALL display a line chart of sentiment trend over the past 4 weeks.
4. THE App SHALL display a pie chart of task status distribution (Todo / In_Progress / Done) across all tasks.
5. WHEN the user selects a different week, THE Analytics_Service SHALL return statistics for the selected week.
6. FOR ALL analytics responses, the total meetings count SHALL equal the count of meetings with status "done" in the specified period for that user (analytics accuracy invariant).
7. FOR ALL analytics responses, the total meeting hours value SHALL equal the sum of duration_seconds for all meetings in the period divided by 3600, rounded to two decimal places (hours calculation invariant).
8. WHERE the user is on the Free_Plan, THE Analytics_Service SHALL return data for the current week only; historical analytics beyond 7 days SHALL require Pro_Plan or Team_Plan.

---

### Requirement 17: Pre-Meeting Push Notification

**User Story:** As a user, I want to receive a push notification 15 minutes before a scheduled meeting with a brief agenda and my pending tasks, so that I can prepare effectively.

#### Acceptance Criteria

1. WHEN a meeting is scheduled in the system, THE Notification_Service SHALL schedule a push notification to be delivered 15 minutes before the meeting start time.
2. WHEN the push notification is delivered, THE App SHALL display the meeting title, scheduled start time, and a count of pending Tasks associated with that meeting.
3. WHEN the user taps the push notification, THE App SHALL open the Home Dashboard screen (S02) or the relevant meeting detail.
4. IF the user has disabled push notifications in device settings, THEN THE Notification_Service SHALL not attempt to deliver notifications and SHALL not display an error.
5. WHEN a meeting is deleted before its scheduled start time, THE Notification_Service SHALL cancel the corresponding scheduled notification.
6. THE Notification_Service SHALL use Expo Notifications for cross-platform push delivery on both iOS and Android.

---

### Requirement 18: AI Chat — Ask a Meeting (RAG)

**User Story:** As a user, I want to ask natural language questions about my past meetings and receive accurate, sourced answers, so that I can quickly retrieve information without manually searching transcripts.

#### Acceptance Criteria

1. WHEN the user submits a question via the AI Chat screen (S09), THE RAG_Service SHALL perform a cosine similarity search against all Embeddings belonging to the authenticated User to retrieve the top-k most relevant Transcript Chunks.
2. WHEN relevant Chunks are retrieved, THE RAG_Service SHALL construct a prompt containing the question and the retrieved Chunks and submit it to the LLM.
3. THE RAG_Service SHALL stream the LLM response back to the App using Server-Sent Events (SSE), displaying tokens as they arrive.
4. WHEN the LLM response is complete, THE App SHALL display the full answer in the chat interface with a reference to the source meeting(s).
5. IF no relevant Chunks are found for a question, THEN THE RAG_Service SHALL return a response indicating that no relevant meeting content was found, rather than hallucinating an answer.
6. WHEN a user on the Free_Plan accesses the AI Chat endpoint, THE Backend SHALL return HTTP 403 Forbidden; AI Chat is available on Pro_Plan and Team_Plan only.
7. FOR ALL AI chat queries, the Embedding_Service SHALL encode the question using the same text-embedding-3-small model used to encode Transcript Chunks, ensuring vector space compatibility (embedding consistency invariant).
8. FOR ALL Transcript Chunks stored in pgvector, the vector dimension SHALL be exactly 1536 (embedding dimension invariant).
9. FOR ALL valid Transcript Chunks, encoding a chunk to an Embedding and then retrieving the nearest neighbour from pgvector SHALL return the original chunk as the top result (round-trip retrieval property).

---

### Requirement 19: Embeddings Pipeline

**User Story:** As a system, I want transcript chunks to be automatically embedded and stored after each meeting, so that the AI chat feature can perform accurate semantic search over all past meetings.

#### Acceptance Criteria

1. WHEN a meeting summarization completes, THE Embedding_Service SHALL split the full Transcript into overlapping Chunks of approximately 500 tokens with a 50-token overlap.
2. WHEN Chunks are created, THE Embedding_Service SHALL call the text-embedding-3-small model to generate a 1536-dimensional vector for each Chunk.
3. THE Embedding_Service SHALL store each Chunk's text and its vector in the pgvector embeddings table, associated with the transcript_id.
4. IF the embedding API call fails for a Chunk, THEN THE Embedding_Service SHALL retry up to 3 times with exponential backoff before logging the failure and continuing with remaining Chunks.
5. FOR ALL stored Embeddings, the vector dimension SHALL be exactly 1536 (dimension invariant).
6. FOR ALL stored Embeddings, the chunk_text field SHALL be non-empty and SHALL correspond to a contiguous segment of the source Transcript (content integrity invariant).
7. THE Embedding_Service SHALL process all Chunks for a meeting asynchronously via Celery so as not to block the summarization response.

---

### Requirement 20: Integration Exports (Slack, Notion, Trello, Email)

**User Story:** As a user, I want to export a meeting summary to Slack, Notion, Trello, or email with a single tap, so that I can share meeting outcomes with my team without manual copy-pasting.

#### Acceptance Criteria

1. WHEN the user taps an export button on the Meeting Summary screen, THE Integration_Service SHALL format the meeting Summary, Decisions, and Tasks into the target platform's required payload structure.
2. WHEN exporting to Slack, THE Integration_Service SHALL post the formatted summary to the configured Slack channel via the Slack Incoming Webhooks API.
3. WHEN exporting to Notion, THE Integration_Service SHALL create a new Notion page in the configured database with the summary content via the Notion API.
4. WHEN exporting to Trello, THE Integration_Service SHALL create a new Trello card in the configured board and list with the summary as the card description via the Trello API.
5. WHEN exporting via email, THE Integration_Service SHALL send the formatted summary to the user's registered email address.
6. IF an export fails due to an API error or invalid credentials, THEN THE Integration_Service SHALL return an error message to the App describing the failure without exposing raw API error details.
7. WHEN an export succeeds, THE Integration_Service SHALL return HTTP 200 and a confirmation message.
8. WHERE the user is on the Free_Plan, THE Integration_Service SHALL reject export requests with HTTP 403 Forbidden; integrations are available on Pro_Plan and Team_Plan only.

---

### Requirement 21: Subscription Plan Enforcement

**User Story:** As the system, I want to enforce subscription tier limits consistently, so that free users cannot exceed their quota and paid features are protected.

#### Acceptance Criteria

1. WHILE a user is on the Free_Plan, THE Plan_Enforcer SHALL reject any meeting start request that would cause the user to exceed 5 meetings in the current calendar month, returning HTTP 403 with a clear upgrade prompt.
2. WHILE a user is on the Free_Plan, THE Plan_Enforcer SHALL stop recording and return an error if a single meeting exceeds 30 minutes in duration.
3. WHILE a user is on the Free_Plan, THE Plan_Enforcer SHALL restrict meeting history retrieval to meetings created within the last 7 days.
4. WHILE a user is on the Free_Plan, THE Plan_Enforcer SHALL block access to AI Chat, Integration exports, and Team Mood Heatmap, returning HTTP 403 for each.
5. WHILE a user is on the Pro_Plan, THE Plan_Enforcer SHALL allow unlimited meetings, unlimited duration, all individual features, and 1-year meeting history.
6. WHILE a user is on the Team_Plan, THE Plan_Enforcer SHALL allow all Pro_Plan features plus shared task board, team mood dashboard, admin controls, and API access for up to 10 users.
7. FOR ALL plan enforcement checks, the enforcement decision SHALL be based on the user's current plan at the time of the request, not the plan at the time of account creation (real-time enforcement invariant).
8. WHEN a user upgrades from Free_Plan to Pro_Plan, THE Plan_Enforcer SHALL immediately grant access to Pro features without requiring a logout or app restart.


### Requirement 22: Home Dashboard

**User Story:** As a user, I want a home screen that shows today's meeting stats and recent meetings, so that I can quickly orient myself and start a new meeting.

#### Acceptance Criteria

1. WHEN the Home Dashboard screen (S02) is opened, THE App SHALL display the count of meetings recorded today, total meeting time today, and a list of the 5 most recent meetings.
2. THE App SHALL display a prominent Start Meeting floating action button (FAB) on the Home Dashboard.
3. WHEN the user taps a recent meeting in the list, THE App SHALL navigate to the Meeting Summary screen (S04) for that meeting.
4. WHEN the user taps the Start Meeting FAB, THE App SHALL initiate the meeting start flow as defined in Requirement 5.
5. THE App SHALL refresh the Home Dashboard data when the screen comes into focus after returning from another screen.
6. IF no meetings have been recorded, THE App SHALL display an empty state with a prompt to start the first meeting.

---

### Requirement 23: Settings Screen

**User Story:** As a user, I want to manage my profile, notification preferences, subscription plan, and team invite link from a settings screen, so that I can configure the app to my needs.

#### Acceptance Criteria

1. WHEN the Settings screen (S11) is opened, THE App SHALL display the user's name, email address, and current subscription plan.
2. WHEN the user updates their display name, THE Auth_Service SHALL persist the change and return the updated user profile.
3. WHEN the user taps the notification toggle, THE App SHALL update the notification preference and THE Notification_Service SHALL respect this preference for all future notifications.
4. WHERE the user is on the Team_Plan, THE App SHALL display a team invite link that can be copied and shared to add members to the team.
5. WHEN the user taps the upgrade plan option, THE App SHALL navigate to the subscription management flow.
6. WHEN the user taps log out, THE App SHALL clear the JWT and Refresh_Token from SecureStore and navigate to the Onboarding screen (S01).

---

### Requirement 24: Data Privacy and Audio Handling

**User Story:** As a user, I want confidence that my meeting audio is not permanently stored, so that sensitive conversations remain private.

#### Acceptance Criteria

1. WHEN a Chunk is processed by THE Transcription_Service, THE Backend SHALL discard the raw audio data immediately after the transcript text is extracted and SHALL NOT persist audio to any storage system.
2. WHEN a meeting ends and summarization is complete, THE Backend SHALL confirm that no audio files for that meeting exist in Cloudflare R2 storage.
3. THE App SHALL display a clear privacy notice during onboarding explaining that audio is processed in real time and not stored.
4. THE Backend SHALL transmit all audio data over encrypted WebSocket connections (WSS) using TLS 1.2 or higher.
5. THE Backend SHALL transmit all REST API data over HTTPS using TLS 1.2 or higher.
6. FOR ALL completed meetings, a query to Cloudflare R2 for audio files associated with that meeting_id SHALL return no results (audio deletion invariant).

---

### Requirement 25: WebSocket Reliability and Reconnection

**User Story:** As a user, I want the live transcription to recover automatically from network interruptions, so that I do not lose meeting content due to connectivity issues.

#### Acceptance Criteria

1. WHEN the WebSocket connection is dropped during a recording, THE App SHALL detect the disconnection within 5 seconds.
2. WHEN a disconnection is detected, THE App SHALL buffer all subsequent audio Chunks in local device memory.
3. WHEN the WebSocket connection is re-established, THE App SHALL transmit buffered Chunks to the Backend in the original chronological order.
4. THE App SHALL attempt WebSocket reconnection with exponential backoff: first retry after 1 second, second after 2 seconds, third after 4 seconds, up to a maximum of 30 seconds between retries.
5. WHEN the WebSocket reconnects and buffered Chunks are transmitted, THE Transcription_Service SHALL process them and return transcripts that are appended to the existing partial transcript in the correct order.
6. FOR ALL reconnection scenarios, the final Transcript stored for a meeting SHALL contain the same content as if no disconnection had occurred (reconnection completeness invariant).
7. FOR ALL reconnection scenarios, no Chunk SHALL be processed more than once by THE Transcription_Service (no-duplicate-processing invariant).

---

### Requirement 26: Rate Limiting and API Protection

**User Story:** As the system, I want to enforce rate limits on API endpoints, so that the service remains available and protected from abuse.

#### Acceptance Criteria

1. THE Backend SHALL enforce a rate limit of 100 requests per minute per authenticated user across all REST endpoints.
2. WHEN a user exceeds the rate limit, THE Backend SHALL return HTTP 429 Too Many Requests with a Retry-After header indicating when the limit resets.
3. THE Backend SHALL enforce a rate limit of 10 requests per minute per IP address on the public authentication endpoints (register, login, Google OAuth).
4. THE Backend SHALL use Redis to store rate limit counters with a sliding window algorithm.
5. FOR ALL rate limit windows, the counter SHALL reset to zero after the window expires (rate limit reset invariant).
6. WHEN a rate-limited request is retried after the Retry-After period, THE Backend SHALL process the request normally.

---

### Requirement 27: Error Handling and Resilience

**User Story:** As a user, I want the app to handle errors gracefully and provide clear feedback, so that I am never left confused about what went wrong.

#### Acceptance Criteria

1. WHEN any API request fails with a network error, THE App SHALL display a user-friendly error message and offer a retry option.
2. WHEN the LLM summarization fails after all retries, THE Meeting_Service SHALL set the meeting status to "failed" and THE App SHALL display an error message with a manual retry button.
3. WHEN the Whisper transcription fails for a Chunk, THE Transcription_Service SHALL log the error, skip the failed Chunk, and continue processing subsequent Chunks.
4. IF the Backend returns HTTP 500, THEN THE App SHALL display a generic error message without exposing internal error details to the user.
5. THE Backend SHALL log all errors with a unique request_id, timestamp, endpoint, and error message to enable debugging.
6. WHEN the App is in an offline state, THE App SHALL display an offline banner and disable features that require network connectivity, while keeping locally cached data accessible.

---

### Requirement 28: Performance Requirements

**User Story:** As a user, I want the app to be fast and responsive, so that it does not interrupt my workflow during meetings.

#### Acceptance Criteria

1. THE Transcription_Service SHALL return a partial transcript for each 30-second audio Chunk within 2 seconds of receiving the Chunk under normal load conditions.
2. THE Summarization_Service SHALL complete the full AI summarization pipeline (summary + decisions + tasks) within 15 seconds for a Transcript representing up to 30 minutes of speech.
3. THE Backend SHALL respond to all REST API requests (excluding AI endpoints) within 500 milliseconds at the 95th percentile under normal load.
4. THE App SHALL render the Home Dashboard screen within 2 seconds of navigation on a mid-range Android device (equivalent to Snapdragon 665 or better).
5. THE App SHALL support concurrent WebSocket connections for at least 100 simultaneous active meetings without degradation in transcription latency.
6. THE Embedding_Service SHALL generate and store Embeddings for a 30-minute meeting Transcript within 60 seconds of summarization completion.

---

### Requirement 29: Onboarding Flow

**User Story:** As a new user, I want a smooth onboarding experience that explains the app's value and gets me set up quickly, so that I can start recording my first meeting within minutes.

#### Acceptance Criteria

1. WHEN the App is launched for the first time with no stored JWT, THE App SHALL display the Onboarding screen (S01) with options for email/password registration and Google sign-in.
2. THE Onboarding screen SHALL display a brief privacy notice explaining that meeting audio is processed in real time and not stored permanently.
3. WHEN registration or sign-in is successful, THE App SHALL navigate to the Home Dashboard screen (S02).
4. THE App SHALL request microphone permission during or immediately after onboarding, before the user attempts to start their first meeting.
5. WHEN the user completes onboarding, THE App SHALL display a brief tutorial overlay on the Home Dashboard highlighting the Start Meeting FAB.

---

### Requirement 30: Team Management (Team Plan)

**User Story:** As a team admin on the Team Plan, I want to invite team members and manage the shared workspace, so that my whole team benefits from shared task boards and mood insights.

#### Acceptance Criteria

1. WHEN a Team_Plan user generates a team invite link, THE Backend SHALL create a unique, time-limited invite token valid for 7 days.
2. WHEN a new or existing user registers or logs in via a valid invite link, THE Backend SHALL add that user to the inviting user's team, up to a maximum of 10 members.
3. WHEN the team reaches 10 members, THE Backend SHALL reject additional invite link registrations with HTTP 403 and a message indicating the team is full.
4. WHILE a user is a team member, THE Task_Service SHALL make all Tasks from all team members visible on the shared task board.
5. WHILE a user is a team member, THE Mood_Service SHALL include that user's Mood_Scores in the team mood heatmap visible to the team admin.
6. WHEN a team admin removes a member, THE Backend SHALL revoke that member's access to shared team data while preserving their individual meeting data.
7. THE Backend SHALL enforce that only the team admin (the user who created the team) can remove members or generate new invite links.


---

## Non-Functional Requirements

### NFR-1: Security

1. THE Auth_Service SHALL hash all passwords using bcrypt with a minimum cost factor of 12 before storage.
2. THE Backend SHALL sign all JWTs using the HS256 algorithm with a secret key of at least 256 bits stored as an environment variable.
3. THE Backend SHALL validate the JWT signature and expiry on every authenticated request before processing it.
4. THE Backend SHALL sanitize all user-supplied string inputs to prevent SQL injection and XSS attacks.
5. THE Backend SHALL enforce HTTPS/WSS for all client-server communication; HTTP and WS connections SHALL be rejected or redirected.
6. THE App SHALL clear all tokens from SecureStore on logout and SHALL NOT cache sensitive data in unencrypted local storage.

### NFR-2: Scalability

1. THE Backend SHALL be deployable as a Docker container and SHALL support horizontal scaling by running multiple Uvicorn worker instances behind a load balancer.
2. THE Celery worker pool SHALL be independently scalable from the FastAPI web workers to handle spikes in summarization demand.
3. THE Backend database connection pool SHALL be configurable via environment variables to support varying load levels.

### NFR-3: Observability

1. THE Backend SHALL emit structured JSON logs for every request, including request_id, user_id, endpoint, HTTP method, status code, and response time.
2. THE Backend SHALL expose a /health endpoint returning HTTP 200 with a JSON body indicating the status of the database, Redis, and Celery connections.
3. THE Backend SHALL track and expose metrics for: active WebSocket connections, Celery queue depth, average transcription latency, and average summarization latency.

### NFR-4: Testability

1. THE Backend SHALL achieve at least 80% line coverage across all service modules as measured by Pytest with coverage reporting.
2. THE Backend SHALL include integration tests for all API endpoints using Pytest and httpx with an in-memory test database.
3. THE App SHALL include unit tests for all Zustand store actions and React Query hooks.

### NFR-5: Accessibility

1. THE App SHALL support dynamic text sizing using the device's accessibility font scale settings.
2. THE App SHALL provide descriptive accessibility labels for all interactive elements (buttons, icons, charts) to support screen readers.
3. THE App SHALL maintain a minimum colour contrast ratio of 4.5:1 for all text elements as per WCAG 2.1 AA guidelines.

### NFR-6: Offline Capability

1. WHILE the device has no network connectivity, THE App SHALL allow users to start a meeting and capture audio using the on-device Whisper base model.
2. WHILE offline, THE App SHALL display cached meeting history and task data from the most recent successful sync.
3. WHEN network connectivity is restored, THE App SHALL automatically sync any offline-captured Transcripts to the Backend and trigger the summarization pipeline.

---

## Correctness Properties for Property-Based Testing

The following properties are derived from the acceptance criteria above and are suitable for implementation as property-based tests using a framework such as Hypothesis (Python) or fast-check (TypeScript/JavaScript).

### P1: Registration Round-Trip (Requirement 1, AC7)
For all valid inputs (name: non-empty string, email: valid RFC 5322 address, password: string of length ≥ 8):
`register(name, email, password)` returns a JWT that is accepted by `GET /meetings` without a 401 error.

### P2: Login Round-Trip (Requirement 2, AC8)
For all registered users:
`login(email, password)` → `use_access_token()` → `refresh_token()` → `use_new_access_token()` succeeds at every step without re-authentication.

### P3: Meeting ID Uniqueness (Requirement 5, AC5)
For any sequence of N meeting start requests by any set of authenticated users:
All returned meeting_ids are distinct (no two meetings share the same ID).

### P4: Transcript Completeness (Requirement 6, AC6)
For any sequence of audio Chunks C1, C2, ..., Cn transmitted for a meeting:
`concat(transcript(C1), transcript(C2), ..., transcript(Cn)) == full_transcript(meeting_id)`

### P5: Summary JSON Structure Invariant (Requirement 7, AC4)
For any valid Transcript T:
`summarize(T)` always returns a JSON object containing non-null fields "summary" (string), "decisions" (array), and "tasks" (array).

### P6: Task Grounding Invariant (Requirement 7, AC7)
For any valid Transcript T and extracted Task set tasks = summarize(T).tasks:
For each task in tasks where task.assignee is non-null:
`task.assignee` appears as a substring or clear reference within T.

### P7: Search Subset Invariant (Requirement 11, AC7)
For any search query Q and user U:
`search(U, Q) ⊆ all_meetings(U)`
The result set of any search is always a subset of the user's complete meeting list.

### P8: Date Filter Correctness (Requirement 11, AC8)
For any date range [start, end] and user U:
For all meetings M in `filter_by_date(U, start, end)`:
`start ≤ M.created_at ≤ end`

### P9: Deletion Cascade Invariant (Requirement 12, AC1, AC5, AC6)
For any meeting M that is deleted:
After deletion, queries for transcripts, summaries, tasks, moods, and embeddings associated with M.id all return empty results.

### P10: Kanban Partition Invariant (Requirement 13, AC5)
For any user U:
`count(tasks_todo(U)) + count(tasks_in_progress(U)) + count(tasks_done(U)) == count(all_tasks(U))`

### P11: Task Status Update Idempotence (Requirement 13, AC7)
For any Task T with current status S:
`update_status(T, S)` followed by `get_task(T)` returns a task with status S, identical to the result of a single `update_status(T, S)` call.

### P12: Mood Score Range Invariant (Requirement 14, AC6)
For all Mood_Score records M stored in the database:
`1 ≤ M.score ≤ 5` and `M.score` is an integer.

### P13: Analytics Meetings Count Invariant (Requirement 16, AC6)
For any user U and time period P:
`analytics(U, P).meetings_count == count(meetings where user_id=U and status='done' and created_at in P)`

### P14: Analytics Hours Calculation Invariant (Requirement 16, AC7)
For any user U and time period P:
`analytics(U, P).total_hours == sum(M.duration_seconds for M in meetings(U, P)) / 3600`
(rounded to 2 decimal places)

### P15: Embedding Dimension Invariant (Requirements 18, 19)
For all Embedding records E stored in pgvector:
`len(E.vector) == 1536`

### P16: Embedding Round-Trip Retrieval (Requirement 18, AC9)
For any Transcript Chunk C that has been embedded and stored:
`nearest_neighbour(embed(C)) == C`
The nearest neighbour search using the chunk's own embedding returns the chunk itself as the top result.

### P17: Embedding Consistency (Requirement 18, AC7)
For any question Q submitted to the RAG_Service:
The embedding of Q is generated using the same text-embedding-3-small model and produces a 1536-dimensional vector, ensuring it lies in the same vector space as stored Transcript Chunk embeddings.

### P18: Rate Limit Reset Invariant (Requirement 26, AC5)
For any user U and rate limit window W of duration D:
After W expires (time > W.start + D), the request counter for U resets to 0 and the next request is processed normally.

### P19: Free Plan Meeting Count Enforcement (Requirement 21, AC1)
For any Free_Plan user U who has recorded exactly 5 meetings in the current calendar month:
`start_meeting(U)` returns HTTP 403.
For any Free_Plan user U who has recorded fewer than 5 meetings in the current calendar month:
`start_meeting(U)` returns HTTP 200 with a valid meeting_id.

### P20: Audio Deletion Invariant (Requirement 24, AC6)
For any completed meeting M:
A query to Cloudflare R2 for objects with key prefix `audio/{M.id}/` returns an empty result set.

### P21: Reconnection Completeness Invariant (Requirement 25, AC6)
For any meeting M where a WebSocket disconnection occurred during recording:
`full_transcript(M) == transcript_that_would_have_been_produced_without_disconnection(M)`
The final stored Transcript is identical regardless of whether disconnections occurred.

### P22: No Duplicate Chunk Processing (Requirement 25, AC7)
For any Chunk C transmitted during a reconnection scenario:
C appears exactly once in the final Transcript for the meeting, regardless of how many times it was buffered or retransmitted.

