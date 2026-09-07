# High-Performance Distributed Media Analytics Platform

# Data Model & Database Design

**Document:** `03-data-model.md`
**Version:** 1.0
**Status:** Draft / Implementation Ready
**Database:** PostgreSQL
**Author:** Adarsh Kumar
**Last Updated:** 2026-09-07

---

# 1. Purpose

This document defines the persistent data model for the High-Performance Distributed Media Analytics Platform.

PostgreSQL acts as the system's **control-plane database**.

It stores:

* users
* authentication sessions
* media metadata
* media assets
* processing jobs
* job dependencies
* job attempts
* job events
* processing manifests
* processing results
* transcripts
* transcript segments
* detections
* scenes
* generated clips
* AI insights
* worker registration and health
* audit events
* reliable outbound events

Large binary objects such as videos, audio files, thumbnails, waveform files and generated media are **not stored directly in PostgreSQL**.

They are stored in object storage.

```text
                    ┌──────────────────────┐
                    │      PostgreSQL      │
                    │                      │
                    │ Users                │
                    │ Media                │
                    │ Jobs                 │
                    │ Job Attempts         │
                    │ Results              │
                    │ Transcripts          │
                    │ Detections           │
                    │ Scenes               │
                    │ Clips                │
                    │ Workers              │
                    │ Audit                │
                    └──────────┬───────────┘
                               │
                       metadata/reference
                               │
                               ▼
                    ┌──────────────────────┐
                    │    Object Storage    │
                    │                      │
                    │ Original Video       │
                    │ Audio                │
                    │ Proxy                │
                    │ Thumbnail             │
                    │ Waveform              │
                    │ Clips                 │
                    │ Subtitles             │
                    └──────────────────────┘
```

---

# 2. Database Design Principles

The database follows these principles.

## 2.1 PostgreSQL is the source of truth

Redis is used for asynchronous job coordination.

PostgreSQL remains the authoritative source for:

* job state
* ownership
* media metadata
* processing results
* audit history
* worker state

A Redis message must never become the only record of a job.

---

# 2.2 Binary data is external

Do not store:

```text
video bytes
audio bytes
thumbnail bytes
large model files
generated clips
```

inside PostgreSQL.

Instead:

```text
PostgreSQL
    │
    └── bucket + object_key
                    │
                    ▼
              Object Storage
```

---

# 2.3 UUID primary keys

Entities use UUID identifiers.

This provides:

* globally unique identifiers
* safe distributed creation
* easier service separation
* reduced dependency on database sequences

The DDL uses PostgreSQL's `gen_random_uuid()` for portability.

The application can later generate UUIDv7 identifiers if time-sortable IDs are desired.

---

# 2.4 Timestamps

All timestamps use:

```sql
TIMESTAMPTZ
```

Never use application-local timestamps for persisted events.

The database stores UTC timestamps.

---

# 2.5 JSONB for extensibility

Structured information that changes frequently between algorithms is stored using:

```sql
JSONB
```

Examples:

```text
model parameters
FFmpeg metadata
AI output
worker capabilities
algorithm-specific metadata
processing options
```

Frequently queried business data remains normalized.

---

# 2.6 Immutable processing history

Processing history should not be destroyed when a new processing run occurs.

For example:

```text
Job #1 → failed
Job #2 → succeeded
Job #3 → succeeded
```

All three jobs remain available for debugging and audit.

---

# 3. Logical Entity Model

The major entities are:

```text
User
 │
 ├── Sessions
 │
 └── Media
       │
       ├── Media Assets
       │
       ├── Jobs
       │     │
       │     ├── Dependencies
       │     ├── Attempts
       │     ├── Events
       │     ├── Results
       │     └── Manifest
       │
       ├── Transcripts
       │     └── Transcript Segments
       │
       ├── Detections
       │
       ├── Scenes
       │
       ├── Clips
       │
       └── AI Insights


Workers
 │
 └── Job Attempts


Audit Logs
Outbox Events
```

---

# 4. Entity Inventory

| Entity                 | Purpose                          |
| ---------------------- | -------------------------------- |
| `users`                | Application users                |
| `user_sessions`        | Refresh/session management       |
| `media`                | Logical media records            |
| `media_assets`         | Physical files in object storage |
| `jobs`                 | Processing tasks                 |
| `job_dependencies`     | Job dependency graph             |
| `job_attempts`         | Individual worker attempts       |
| `job_events`           | Immutable job lifecycle events   |
| `processing_manifests` | C++/Python processing contract   |
| `processing_results`   | Generic processing result        |
| `transcripts`          | Audio transcription results      |
| `transcript_segments`  | Timestamped transcript chunks    |
| `detections`           | Faces/objects/entities detected  |
| `scenes`               | Video scene boundaries           |
| `clips`                | Generated video clips            |
| `ai_insights`          | AI-generated analysis            |
| `workers`              | Worker registration              |
| `audit_logs`           | Security/business audit trail    |
| `outbox_events`        | Reliable event publication       |

---

# 5. Users

## 5.1 Purpose

The `users` table represents authenticated application users.

```text
users
  │
  ├── media
  ├── sessions
  └── audit_logs
```

Important rule:

**Never store plaintext passwords.**

Only password hashes are stored.

---

# 6. User Sessions

`user_sessions` stores server-side session/refresh-token metadata.

A token itself should not be stored.

Instead:

```text
refresh token
       │
       ▼
SHA-256 / secure hash
       │
       ▼
token_hash
```

This means a database compromise does not directly expose active refresh tokens.

---

# 7. Media

`media` represents the logical media object.

Example:

```text
Media
├── title
├── original filename
├── media type
├── duration
├── resolution
├── codec information
└── metadata
```

It does **not** contain the actual video bytes.

---

# 8. Media Assets

A single logical media object can have multiple physical assets.

Example:

```text
Media
 │
 ├── original.mp4
 ├── proxy.mp4
 ├── audio.wav
 ├── thumbnail.jpg
 ├── waveform.json
 └── subtitles.vtt
```

Each physical object is represented by `media_assets`.

Asset types include:

```text
source
proxy
audio
thumbnail
waveform
clip
subtitle
manifest
other
```

---

# 9. Jobs

A job represents one asynchronous processing operation.

Examples:

```text
MEDIA_PROBE
VIDEO_TRANSCODE
AUDIO_EXTRACT
THUMBNAIL_GENERATE
SCENE_DETECT
FACE_DETECT
TRANSCRIBE
SENTIMENT_ANALYSIS
SUMMARY
CLIP_GENERATE
```

Job state:

```text
queued
   │
   ▼
leased
   │
   ▼
running
   │
 ┌─┴─────────────┐
 ▼               ▼
succeeded       failed
                 │
                 ▼
          retry / dead_letter
```

Jobs contain:

* priority
* retry information
* lease information
* worker assignment
* input parameters
* output information
* error information
* correlation ID

---

# 10. Job Dependencies

Some processing operations require previous operations.

Example:

```text
MEDIA_PROBE
     │
     ├──────────────┐
     ▼              ▼
AUDIO_EXTRACT    THUMBNAIL
     │
     ▼
TRANSCRIBE
     │
     ▼
SUMMARY
```

`job_dependencies` represents this graph.

A dependency means:

```text
prerequisite_job
       │
       ▼
dependent_job
```

---

# 11. Job Attempts

A job can execute multiple times.

Example:

```text
Job #100

Attempt 1 → C++ worker → timeout
Attempt 2 → C++ worker → crash
Attempt 3 → C++ worker → success
```

The `jobs.attempt_count` field provides the current aggregate.

`job_attempts` stores the detailed history.

---

# 12. Job Events

`job_events` is an append-only lifecycle history.

Example:

```text
created
queued
leased
started
progress
completed
failed
retried
cancelled
```

This is useful for:

* debugging
* audit
* WebSocket updates
* monitoring
* timeline visualization

---

# 13. Processing Manifest

The processing manifest is the contract between workers.

Example:

```json
{
  "schema_version": 1,
  "media": {
    "duration_ms": 124500,
    "width": 1920,
    "height": 1080
  },
  "input": {
    "bucket": "media",
    "object_key": "media/123/source.mp4"
  },
  "outputs": [
    "audio",
    "thumbnail",
    "proxy"
  ]
}
```

The manifest allows C++ and Python services to communicate without sharing process memory.

---

# 14. Processing Results

`processing_results` provides a generic result envelope.

Specialized results are stored in dedicated tables.

Example:

```text
processing_results
       │
       ├── transcript
       ├── detection
       ├── scene
       ├── clip
       └── AI insight
```

---

# 15. Transcripts

A transcript represents one transcription run.

Fields include:

* language
* model
* full text
* confidence
* processing job
* creation timestamp

---

# 16. Transcript Segments

A transcript is divided into timestamped segments.

Example:

```text
00:00.000 ───────── 00:04.200
"Welcome to the platform."

00:04.200 ───────── 00:08.500
"Today we will analyze this video."
```

Each segment can contain:

* start time
* end time
* text
* speaker
* confidence
* word-level information

---

# 17. Detections

The detection model supports:

```text
face
person
object
logo
vehicle
animal
custom
```

Each detection can contain:

```text
label
confidence
timestamp
frame number
bounding box
track ID
attributes
```

Bounding box coordinates are normalized or represented relative to the source frame.

---

# 18. Scenes

Scenes represent detected video segments.

Example:

```text
Scene 0
00:00 → 00:12

Scene 1
00:12 → 00:38

Scene 2
00:38 → 01:04
```

Scene boundaries can be generated by C++ processing or an AI model.

---

# 19. Clips

A clip represents a selected or generated portion of media.

Example:

```text
Original
00:00 ───────────────────────── 10:00

Clip
         02:10 ───── 02:48
```

The resulting physical clip is represented using `media_assets`.

---

# 20. AI Insights

AI insights are intentionally flexible.

Supported types can include:

```text
summary
sentiment
keywords
topics
moderation
classification
embedding
speaker_analysis
```

The actual model output is stored as JSONB.

---

# 21. Workers

Workers register themselves in PostgreSQL.

Example:

```text
worker-cpp-01
worker-cpp-02
worker-python-01
worker-python-02
```

Worker metadata includes:

* worker type
* version
* hostname
* capabilities
* status
* heartbeat
* startup time

---

# 22. Worker Lifecycle

```text
offline
   │
   ▼
online
   │
   ▼
draining
   │
   ▼
offline
```

A worker that stops sending heartbeats can be considered unhealthy.

---

# 23. Audit Logs

Audit logs track security-sensitive and business-sensitive actions.

Examples:

```text
USER_LOGIN
USER_LOGOUT
MEDIA_CREATED
MEDIA_DELETED
JOB_CANCELLED
MEDIA_DOWNLOADED
PERMISSION_CHANGED
```

Audit logs are separate from processing events.

---

# 24. Outbox Events

The outbox provides reliable event publication.

Example:

```text
PostgreSQL transaction
        │
        ├── create job
        │
        └── create outbox event
                 │
                 ▼
           Outbox Publisher
                 │
                 ▼
               Redis
```

This prevents a situation where:

```text
DB transaction succeeds
       +
Redis publish fails
       =
job exists but is never queued
```

---

# 25. Relationship Summary

```text
users
  │
  ├────< user_sessions
  │
  └────< media
            │
            ├────< media_assets
            │
            ├────< jobs
            │        │
            │        ├────< job_attempts
            │        ├────< job_events
            │        ├────< processing_results
            │        └────< processing_manifests
            │
            ├────< transcripts
            │        │
            │        └────< transcript_segments
            │
            ├────< detections
            ├────< scenes
            ├────< clips
            └────< ai_insights

workers
  │
  └────< job_attempts
```

---

# 26. Complete PostgreSQL DDL

The following schema is designed to run on a fresh PostgreSQL database.

Save it as:

```text
database/schema.sql
```

Then execute:

```bash
psql "$DATABASE_URL" -f database/schema.sql
```

---

## 26.1 Extensions

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

---

## 26.2 Updated-at Function

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;
```

---

# 27. Users

```sql
CREATE TABLE users
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    email TEXT NOT NULL,

    password_hash TEXT NOT NULL,

    display_name TEXT NOT NULL,

    role TEXT NOT NULL DEFAULT 'user',

    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    last_login_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT users_role_check
        CHECK (role IN ('user', 'admin')),

    CONSTRAINT users_display_name_check
        CHECK (length(trim(display_name)) BETWEEN 1 AND 150)
);
```

Email uniqueness is case-insensitive:

```sql
CREATE UNIQUE INDEX users_email_unique_idx
    ON users (LOWER(email));
```

---

# 28. User Sessions

```sql
CREATE TABLE user_sessions
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    user_id UUID NOT NULL,

    token_hash TEXT NOT NULL,

    user_agent TEXT,

    ip_address INET,

    expires_at TIMESTAMPTZ NOT NULL,

    revoked_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT user_sessions_user_fk
        FOREIGN KEY (user_id)
        REFERENCES users(id)
        ON DELETE CASCADE,

    CONSTRAINT user_sessions_token_unique
        UNIQUE (token_hash)
);

CREATE INDEX user_sessions_user_idx
    ON user_sessions(user_id);

CREATE INDEX user_sessions_expiry_idx
    ON user_sessions(expires_at);
```

---

# 29. Media

```sql
CREATE TABLE media
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    owner_id UUID NOT NULL,

    title TEXT NOT NULL,

    original_filename TEXT NOT NULL,

    media_type TEXT NOT NULL,

    status TEXT NOT NULL DEFAULT 'uploading',

    mime_type TEXT,

    size_bytes BIGINT,

    duration_ms BIGINT,

    width INTEGER,

    height INTEGER,

    frame_rate_num INTEGER,

    frame_rate_den INTEGER,

    sample_rate INTEGER,

    channels INTEGER,

    codec TEXT,

    checksum_sha256 TEXT,

    metadata JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    deleted_at TIMESTAMPTZ,

    CONSTRAINT media_owner_fk
        FOREIGN KEY (owner_id)
        REFERENCES users(id)
        ON DELETE RESTRICT,

    CONSTRAINT media_type_check
        CHECK (media_type IN ('video', 'audio')),

    CONSTRAINT media_status_check
        CHECK
        (
            status IN
            (
                'uploading',
                'ready',
                'processing',
                'processed',
                'failed',
                'deleted'
            )
        ),

    CONSTRAINT media_size_check
        CHECK (size_bytes IS NULL OR size_bytes >= 0),

    CONSTRAINT media_duration_check
        CHECK (duration_ms IS NULL OR duration_ms >= 0),

    CONSTRAINT media_dimensions_check
        CHECK
        (
            (width IS NULL AND height IS NULL)
            OR
            (width > 0 AND height > 0)
        ),

    CONSTRAINT media_frame_rate_check
        CHECK
        (
            (frame_rate_num IS NULL AND frame_rate_den IS NULL)
            OR
            (
                frame_rate_num > 0
                AND frame_rate_den > 0
            )
        ),

    CONSTRAINT media_sample_rate_check
        CHECK (sample_rate IS NULL OR sample_rate > 0),

    CONSTRAINT media_channels_check
        CHECK (channels IS NULL OR channels > 0)
);
```

Indexes:

```sql
CREATE INDEX media_owner_created_idx
    ON media(owner_id, created_at DESC);

CREATE INDEX media_status_created_idx
    ON media(status, created_at DESC);

CREATE INDEX media_active_owner_idx
    ON media(owner_id, created_at DESC)
    WHERE deleted_at IS NULL;
```

---

# 30. Media Assets

```sql
CREATE TABLE media_assets
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    media_id UUID NOT NULL,

    generated_by_job_id UUID,

    asset_type TEXT NOT NULL,

    storage_provider TEXT NOT NULL DEFAULT 's3',

    bucket TEXT NOT NULL,

    object_key TEXT NOT NULL,

    content_type TEXT,

    size_bytes BIGINT,

    checksum_sha256 TEXT,

    duration_ms BIGINT,

    width INTEGER,

    height INTEGER,

    frame_rate_num INTEGER,

    frame_rate_den INTEGER,

    sample_rate INTEGER,

    channels INTEGER,

    codec TEXT,

    metadata JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT media_assets_media_fk
        FOREIGN KEY (media_id)
        REFERENCES media(id)
        ON DELETE CASCADE,

    CONSTRAINT media_assets_asset_job_fk
        FOREIGN KEY (generated_by_job_id)
        REFERENCES jobs(id)
        ON DELETE SET NULL,

    CONSTRAINT media_assets_type_check
        CHECK
        (
            asset_type IN
            (
                'source',
                'proxy',
                'audio',
                'thumbnail',
                'waveform',
                'clip',
                'subtitle',
                'manifest',
                'other'
            )
        ),

    CONSTRAINT media_assets_size_check
        CHECK (size_bytes IS NULL OR size_bytes >= 0),

    CONSTRAINT media_assets_duration_check
        CHECK (duration_ms IS NULL OR duration_ms >= 0)
);
```

The `generated_by_job_id` FK references `jobs`, which must exist first. Therefore, in the final executable schema the `media_assets` table is created **after `jobs`**.

The complete ordering appears in the consolidated schema in Section 42.

---

# 31. Jobs

```sql
CREATE TABLE jobs
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    media_id UUID NOT NULL,

    type TEXT NOT NULL,

    status TEXT NOT NULL DEFAULT 'queued',

    priority INTEGER NOT NULL DEFAULT 100,

    attempt_count INTEGER NOT NULL DEFAULT 0,

    max_attempts INTEGER NOT NULL DEFAULT 3,

    available_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    leased_until TIMESTAMPTZ,

    worker_id UUID,

    correlation_id UUID NOT NULL DEFAULT gen_random_uuid(),

    idempotency_key TEXT,

    input JSONB NOT NULL DEFAULT '{}'::JSONB,

    output JSONB,

    error_code TEXT,

    error_message TEXT,

    error_details JSONB,

    started_at TIMESTAMPTZ,

    completed_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT jobs_media_fk
        FOREIGN KEY (media_id)
        REFERENCES media(id)
        ON DELETE CASCADE,

    CONSTRAINT jobs_worker_fk
        FOREIGN KEY (worker_id)
        REFERENCES workers(id)
        ON DELETE SET NULL,

    CONSTRAINT jobs_status_check
        CHECK
        (
            status IN
            (
                'queued',
                'leased',
                'running',
                'succeeded',
                'failed',
                'cancelled',
                'dead_letter'
            )
        ),

    CONSTRAINT jobs_priority_check
        CHECK (priority >= 0),

    CONSTRAINT jobs_attempt_count_check
        CHECK (attempt_count >= 0),

    CONSTRAINT jobs_max_attempts_check
        CHECK (max_attempts > 0)
);
```

---

# 32. Job Dependencies

```sql
CREATE TABLE job_dependencies
(
    prerequisite_job_id UUID NOT NULL,

    dependent_job_id UUID NOT NULL,

    dependency_type TEXT NOT NULL DEFAULT 'blocks',

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY
    (
        prerequisite_job_id,
        dependent_job_id
    ),

    CONSTRAINT job_dependencies_prerequisite_fk
        FOREIGN KEY (prerequisite_job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT job_dependencies_dependent_fk
        FOREIGN KEY (dependent_job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT job_dependencies_self_check
        CHECK (prerequisite_job_id <> dependent_job_id),

    CONSTRAINT job_dependencies_type_check
        CHECK (dependency_type IN ('blocks'))
);
```

---

# 33. Job Attempts

```sql
CREATE TABLE job_attempts
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    job_id UUID NOT NULL,

    attempt_number INTEGER NOT NULL,

    worker_id UUID,

    status TEXT NOT NULL DEFAULT 'started',

    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    finished_at TIMESTAMPTZ,

    error_code TEXT,

    error_message TEXT,

    metrics JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT job_attempts_job_fk
        FOREIGN KEY (job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT job_attempts_worker_fk
        FOREIGN KEY (worker_id)
        REFERENCES workers(id)
        ON DELETE SET NULL,

    CONSTRAINT job_attempts_number_check
        CHECK (attempt_number > 0),

    CONSTRAINT job_attempts_status_check
        CHECK
        (
            status IN
            (
                'started',
                'succeeded',
                'failed',
                'timed_out'
            )
        ),

    CONSTRAINT job_attempts_unique
        UNIQUE(job_id, attempt_number)
);
```

---

# 34. Job Events

```sql
CREATE TABLE job_events
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    job_id UUID NOT NULL,

    event_type TEXT NOT NULL,

    progress_percent NUMERIC(5,2),

    message TEXT,

    payload JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT job_events_job_fk
        FOREIGN KEY (job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT job_events_progress_check
        CHECK
        (
            progress_percent IS NULL
            OR
            (
                progress_percent >= 0
                AND progress_percent <= 100
            )
        ),

    CONSTRAINT job_events_type_check
        CHECK
        (
            event_type IN
            (
                'created',
                'queued',
                'leased',
                'started',
                'progress',
                'completed',
                'failed',
                'retried',
                'cancelled',
                'dead_letter'
            )
        )
);
```

---

# 35. Processing Manifests

```sql
CREATE TABLE processing_manifests
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    media_id UUID NOT NULL,

    job_id UUID NOT NULL,

    schema_version INTEGER NOT NULL DEFAULT 1,

    manifest JSONB NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT processing_manifests_media_fk
        FOREIGN KEY (media_id)
        REFERENCES media(id)
        ON DELETE CASCADE,

    CONSTRAINT processing_manifests_job_fk
        FOREIGN KEY (job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT processing_manifests_schema_check
        CHECK (schema_version > 0),

    CONSTRAINT processing_manifests_job_unique
        UNIQUE(job_id, schema_version)
);
```

---

# 36. Processing Results

```sql
CREATE TABLE processing_results
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    job_id UUID NOT NULL,

    result_type TEXT NOT NULL,

    summary JSONB NOT NULL DEFAULT '{}'::JSONB,

    metrics JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT processing_results_job_fk
        FOREIGN KEY (job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT processing_results_type_check
        CHECK
        (
            result_type IN
            (
                'probe',
                'transcode',
                'audio_extract',
                'thumbnail',
                'scene_detection',
                'face_detection',
                'object_detection',
                'transcription',
                'clip_generation',
                'ai_analysis',
                'other'
            )
        )
);
```

---

# 37. Transcripts

```sql
CREATE TABLE transcripts
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    media_id UUID NOT NULL,

    job_id UUID NOT NULL,

    language TEXT,

    model_name TEXT,

    full_text TEXT NOT NULL DEFAULT '',

    confidence NUMERIC(5,4),

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT transcripts_media_fk
        FOREIGN KEY (media_id)
        REFERENCES media(id)
        ON DELETE CASCADE,

    CONSTRAINT transcripts_job_fk
        FOREIGN KEY (job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT transcripts_confidence_check
        CHECK
        (
            confidence IS NULL
            OR
            (
                confidence >= 0
                AND confidence <= 1
            )
        ),

    CONSTRAINT transcripts_job_unique
        UNIQUE(job_id)
);
```

---

# 38. Transcript Segments

```sql
CREATE TABLE transcript_segments
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    transcript_id UUID NOT NULL,

    segment_index INTEGER NOT NULL,

    start_ms BIGINT NOT NULL,

    end_ms BIGINT NOT NULL,

    text TEXT NOT NULL,

    speaker_label TEXT,

    confidence NUMERIC(5,4),

    words JSONB NOT NULL DEFAULT '[]'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT transcript_segments_transcript_fk
        FOREIGN KEY (transcript_id)
        REFERENCES transcripts(id)
        ON DELETE CASCADE,

    CONSTRAINT transcript_segments_index_check
        CHECK (segment_index >= 0),

    CONSTRAINT transcript_segments_time_check
        CHECK
        (
            start_ms >= 0
            AND end_ms > start_ms
        ),

    CONSTRAINT transcript_segments_confidence_check
        CHECK
        (
            confidence IS NULL
            OR
            (
                confidence >= 0
                AND confidence <= 1
            )
        ),

    CONSTRAINT transcript_segments_unique
        UNIQUE(transcript_id, segment_index)
);
```

---

# 39. Detections

```sql
CREATE TABLE detections
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    media_id UUID NOT NULL,

    job_id UUID NOT NULL,

    detection_type TEXT NOT NULL,

    label TEXT NOT NULL,

    confidence NUMERIC(5,4),

    start_ms BIGINT,

    end_ms BIGINT,

    frame_number BIGINT,

    bbox_x NUMERIC(10,6),

    bbox_y NUMERIC(10,6),

    bbox_width NUMERIC(10,6),

    bbox_height NUMERIC(10,6),

    track_id TEXT,

    attributes JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT detections_media_fk
        FOREIGN KEY (media_id)
        REFERENCES media(id)
        ON DELETE CASCADE,

    CONSTRAINT detections_job_fk
        FOREIGN KEY (job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT detections_type_check
        CHECK
        (
            detection_type IN
            (
                'face',
                'person',
                'object',
                'logo',
                'vehicle',
                'animal',
                'custom'
            )
        ),

    CONSTRAINT detections_confidence_check
        CHECK
        (
            confidence IS NULL
            OR
            (
                confidence >= 0
                AND confidence <= 1
            )
        ),

    CONSTRAINT detections_time_check
        CHECK
        (
            (start_ms IS NULL AND end_ms IS NULL)
            OR
            (
                start_ms >= 0
                AND end_ms > start_ms
            )
        ),

    CONSTRAINT detections_bbox_check
        CHECK
        (
            (
                bbox_x IS NULL
                AND bbox_y IS NULL
                AND bbox_width IS NULL
                AND bbox_height IS NULL
            )
            OR
            (
                bbox_x >= 0
                AND bbox_y >= 0
                AND bbox_width >= 0
                AND bbox_height >= 0
            )
        )
);
```

---

# 40. Scenes

```sql
CREATE TABLE scenes
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    media_id UUID NOT NULL,

    job_id UUID NOT NULL,

    scene_index INTEGER NOT NULL,

    start_ms BIGINT NOT NULL,

    end_ms BIGINT NOT NULL,

    score NUMERIC(5,4),

    label TEXT,

    metadata JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT scenes_media_fk
        FOREIGN KEY (media_id)
        REFERENCES media(id)
        ON DELETE CASCADE,

    CONSTRAINT scenes_job_fk
        FOREIGN KEY (job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT scenes_index_check
        CHECK (scene_index >= 0),

    CONSTRAINT scenes_time_check
        CHECK
        (
            start_ms >= 0
            AND end_ms > start_ms
        ),

    CONSTRAINT scenes_score_check
        CHECK
        (
            score IS NULL
            OR
            (
                score >= 0
                AND score <= 1
            )
        ),

    CONSTRAINT scenes_unique
        UNIQUE(job_id, scene_index)
);
```

---

# 41. Clips

```sql
CREATE TABLE clips
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    media_id UUID NOT NULL,

    job_id UUID,

    source_asset_id UUID,

    output_asset_id UUID,

    name TEXT NOT NULL,

    start_ms BIGINT NOT NULL,

    end_ms BIGINT NOT NULL,

    status TEXT NOT NULL DEFAULT 'requested',

    requested_format TEXT,

    metadata JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT clips_media_fk
        FOREIGN KEY (media_id)
        REFERENCES media(id)
        ON DELETE CASCADE,

    CONSTRAINT clips_job_fk
        FOREIGN KEY (job_id)
        REFERENCES jobs(id)
        ON DELETE SET NULL,

    CONSTRAINT clips_source_asset_fk
        FOREIGN KEY (source_asset_id)
        REFERENCES media_assets(id)
        ON DELETE SET NULL,

    CONSTRAINT clips_output_asset_fk
        FOREIGN KEY (output_asset_id)
        REFERENCES media_assets(id)
        ON DELETE SET NULL,

    CONSTRAINT clips_time_check
        CHECK
        (
            start_ms >= 0
            AND end_ms > start_ms
        ),

    CONSTRAINT clips_status_check
        CHECK
        (
            status IN
            (
                'requested',
                'processing',
                'ready',
                'failed',
                'cancelled'
            )
        )
);
```

---

# 42. AI Insights

```sql
CREATE TABLE ai_insights
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    media_id UUID NOT NULL,

    job_id UUID NOT NULL,

    insight_type TEXT NOT NULL,

    model_name TEXT,

    content JSONB NOT NULL,

    confidence NUMERIC(5,4),

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT ai_insights_media_fk
        FOREIGN KEY (media_id)
        REFERENCES media(id)
        ON DELETE CASCADE,

    CONSTRAINT ai_insights_job_fk
        FOREIGN KEY (job_id)
        REFERENCES jobs(id)
        ON DELETE CASCADE,

    CONSTRAINT ai_insights_type_check
        CHECK
        (
            insight_type IN
            (
                'summary',
                'sentiment',
                'keywords',
                'topics',
                'moderation',
                'classification',
                'embedding',
                'speaker_analysis',
                'custom'
            )
        ),

    CONSTRAINT ai_insights_confidence_check
        CHECK
        (
            confidence IS NULL
            OR
            (
                confidence >= 0
                AND confidence <= 1
            )
        )
);
```

---

# 43. Workers

Workers must exist before tables that reference them.

```sql
CREATE TABLE workers
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    worker_key TEXT NOT NULL,

    worker_type TEXT NOT NULL,

    hostname TEXT,

    version TEXT,

    status TEXT NOT NULL DEFAULT 'offline',

    capabilities JSONB NOT NULL DEFAULT '{}'::JSONB,

    last_heartbeat_at TIMESTAMPTZ,

    started_at TIMESTAMPTZ,

    metadata JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT workers_key_unique
        UNIQUE(worker_key),

    CONSTRAINT workers_type_check
        CHECK
        (
            worker_type IN
            (
                'cpp',
                'python'
            )
        ),

    CONSTRAINT workers_status_check
        CHECK
        (
            status IN
            (
                'online',
                'draining',
                'offline'
            )
        )
);
```

---

# 44. Audit Logs

```sql
CREATE TABLE audit_logs
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    actor_user_id UUID,

    action TEXT NOT NULL,

    resource_type TEXT,

    resource_id UUID,

    metadata JSONB NOT NULL DEFAULT '{}'::JSONB,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT audit_logs_actor_fk
        FOREIGN KEY (actor_user_id)
        REFERENCES users(id)
        ON DELETE SET NULL
);
```

---

# 45. Outbox Events

```sql
CREATE TABLE outbox_events
(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    aggregate_type TEXT NOT NULL,

    aggregate_id UUID NOT NULL,

    event_type TEXT NOT NULL,

    payload JSONB NOT NULL,

    published_at TIMESTAMPTZ,

    attempts INTEGER NOT NULL DEFAULT 0,

    last_error TEXT,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT outbox_attempts_check
        CHECK (attempts >= 0)
);
```

---

# 46. Required Indexes

## Jobs

```sql
CREATE INDEX jobs_queue_idx
    ON jobs(priority DESC, available_at ASC)
    WHERE status = 'queued';

CREATE INDEX jobs_media_status_idx
    ON jobs(media_id, status);

CREATE INDEX jobs_worker_status_idx
    ON jobs(worker_id, status);

CREATE INDEX jobs_correlation_idx
    ON jobs(correlation_id);

CREATE INDEX jobs_created_idx
    ON jobs(created_at DESC);

CREATE UNIQUE INDEX jobs_idempotency_unique_idx
    ON jobs(media_id, idempotency_key)
    WHERE idempotency_key IS NOT NULL;
```

---

## Job dependencies

```sql
CREATE INDEX job_dependencies_prerequisite_idx
    ON job_dependencies(prerequisite_job_id);

CREATE INDEX job_dependencies_dependent_idx
    ON job_dependencies(dependent_job_id);
```

---

## Job attempts

```sql
CREATE INDEX job_attempts_job_idx
    ON job_attempts(job_id);

CREATE INDEX job_attempts_worker_idx
    ON job_attempts(worker_id);

CREATE INDEX job_attempts_status_idx
    ON job_attempts(status);
```

---

## Job events

```sql
CREATE INDEX job_events_job_created_idx
    ON job_events(job_id, created_at ASC);
```

---

## Media assets

```sql
CREATE UNIQUE INDEX media_assets_object_unique_idx
    ON media_assets(storage_provider, bucket, object_key);

CREATE INDEX media_assets_media_idx
    ON media_assets(media_id, asset_type);

CREATE INDEX media_assets_job_idx
    ON media_assets(generated_by_job_id);
```

---

## Transcripts

```sql
CREATE INDEX transcripts_media_idx
    ON transcripts(media_id, created_at DESC);

CREATE INDEX transcript_segments_time_idx
    ON transcript_segments(transcript_id, start_ms);
```

---

## Detections

```sql
CREATE INDEX detections_media_time_idx
    ON detections(media_id, start_ms);

CREATE INDEX detections_media_type_idx
    ON detections(media_id, detection_type);

CREATE INDEX detections_track_idx
    ON detections(media_id, track_id);
```

---

## Scenes

```sql
CREATE INDEX scenes_media_time_idx
    ON scenes(media_id, start_ms);
```

---

## Clips

```sql
CREATE INDEX clips_media_idx
    ON clips(media_id, created_at DESC);

CREATE INDEX clips_status_idx
    ON clips(status);
```

---

## AI insights

```sql
CREATE INDEX ai_insights_media_idx
    ON ai_insights(media_id, created_at DESC);

CREATE INDEX ai_insights_type_idx
    ON ai_insights(media_id, insight_type);
```

---

## Workers

```sql
CREATE INDEX workers_status_idx
    ON workers(status);

CREATE INDEX workers_heartbeat_idx
    ON workers(last_heartbeat_at);
```

---

## Audit logs

```sql
CREATE INDEX audit_logs_actor_idx
    ON audit_logs(actor_user_id, created_at DESC);

CREATE INDEX audit_logs_resource_idx
    ON audit_logs(resource_type, resource_id);

CREATE INDEX audit_logs_created_idx
    ON audit_logs(created_at DESC);
```

---

## Outbox

```sql
CREATE INDEX outbox_unpublished_idx
    ON outbox_events(created_at ASC)
    WHERE published_at IS NULL;
```

---

# 47. Updated-at Triggers

```sql
CREATE TRIGGER users_set_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER media_set_updated_at
BEFORE UPDATE ON media
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER workers_set_updated_at
BEFORE UPDATE ON workers
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER clips_set_updated_at
BEFORE UPDATE ON clips
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

---

# 48. Complete Executable Schema

For implementation, the recommended physical ordering is:

```text
1.  pgcrypto
2.  set_updated_at()
3.  users
4.  user_sessions
5.  workers
6.  media
7.  jobs
8.  job_dependencies
9.  job_attempts
10. job_events
11. media_assets
12. processing_manifests
13. processing_results
14. transcripts
15. transcript_segments
16. detections
17. scenes
18. clips
19. ai_insights
20. audit_logs
21. outbox_events
22. indexes
23. triggers
```

This ordering avoids circular foreign-key creation problems.

---

# 49. Seed Data

Development-only seed data:

```sql
INSERT INTO users
(
    email,
    password_hash,
    display_name,
    role
)
VALUES
(
    'admin@media-platform.local',
    '$2b$12$REPLACE_WITH_REAL_BCRYPT_HASH',
    'Development Admin',
    'admin'
);
```

Do not use this password hash in production.

---

# 50. Example Media Record

```sql
INSERT INTO media
(
    owner_id,
    title,
    original_filename,
    media_type,
    status,
    mime_type,
    size_bytes
)
SELECT
    id,
    'Demo Video',
    'demo.mp4',
    'video',
    'ready',
    'video/mp4',
    104857600
FROM users
WHERE email = 'admin@media-platform.local';
```

---

# 51. Example Job

```sql
INSERT INTO jobs
(
    media_id,
    type,
    status,
    priority,
    max_attempts,
    input
)
SELECT
    id,
    'MEDIA_PROBE',
    'queued',
    100,
    3,
    '{"probe": true}'::JSONB
FROM media
WHERE original_filename = 'demo.mp4';
```

---

# 52. Example Processing Flow

A real processing operation should create records in approximately this order:

```text
1. users
      │
      ▼
2. media
      │
      ▼
3. source media_asset
      │
      ▼
4. MEDIA_PROBE job
      │
      ▼
5. job_attempt
      │
      ▼
6. processing_result
      │
      ├── metadata update
      │
      └── generated assets
               │
               ▼
7. AUDIO_EXTRACT job
      │
      ▼
8. TRANSCRIBE job
      │
      ▼
9. transcript
      │
      ▼
10. transcript_segments
      │
      ▼
11. AI_ANALYSIS job
      │
      ▼
12. ai_insights
```

---

# 53. Job Queue and PostgreSQL

The system deliberately separates:

```text
Redis
    =
transport / queue coordination

PostgreSQL
    =
persistent state / source of truth
```

Example:

```text
Node.js
   │
   ├── INSERT jobs
   │
   ├── INSERT outbox_events
   │
   ▼
PostgreSQL
   │
   ▼
Outbox Publisher
   │
   ▼
Redis
   │
   ▼
C++ Worker
   │
   ├── UPDATE jobs
   ├── INSERT job_attempt
   ├── INSERT job_events
   └── INSERT processing_results
```

---

# 54. Job Leasing

A worker should not simply mark a job as running forever.

The job contains:

```text
leased_until
```

Example:

```text
Job
status = running

leased_until =
2026-09-07 22:40:00 UTC
```

If the worker crashes:

```text
leased_until < NOW()
```

the recovery process can identify the abandoned job.

---

# 55. Retry Model

The database supports:

```text
attempt_count
max_attempts
```

Example:

```text
max_attempts = 3

Attempt 1 → failed
Attempt 2 → failed
Attempt 3 → failed

        ↓

dead_letter
```

Retry policy itself remains application logic because different failures have different retry semantics.

---

# 56. Idempotency

Jobs support:

```text
idempotency_key
```

Example:

```text
media_id = ABC
idempotency_key = "transcribe-v1-en"
```

Repeated requests can be detected without creating duplicate processing operations.

---

# 57. Data Retention

Recommended initial retention:

| Data          | Retention                           |
| ------------- | ----------------------------------- |
| Users         | Indefinite                          |
| Sessions      | Until expiry + cleanup              |
| Media         | User-controlled                     |
| Media assets  | Until media deletion                |
| Jobs          | Long-term                           |
| Job events    | Long-term initially                 |
| Job attempts  | Long-term                           |
| Transcripts   | With media                          |
| Detections    | With media                          |
| Scenes        | With media                          |
| Clips         | User-controlled                     |
| AI insights   | With processing run                 |
| Audit logs    | Long-term                           |
| Outbox events | Delete after successful publication |

Production retention can later be changed according to storage requirements.

---

# 58. Data Consistency Rules

## Rule 1

Every job must reference an existing media object.

```text
jobs.media_id → media.id
```

## Rule 2

Every transcript belongs to a media object.

```text
transcripts.media_id → media.id
```

## Rule 3

Every transcript segment belongs to a transcript.

```text
transcript_segments.transcript_id
```

## Rule 4

Every generated asset belongs to a media object.

```text
media_assets.media_id
```

## Rule 5

Worker assignment is nullable.

A job may exist before a worker claims it.

## Rule 6

Processing history is append-oriented.

Do not overwrite historical attempts.

---

# 59. Delete Semantics

The schema intentionally distinguishes between relationships.

### Cascade

Used when child data has no meaning without the parent.

Example:

```text
media
 └── transcripts
      └── transcript_segments
```

Deleting media removes its dependent processing metadata.

### Set NULL

Used when historical data can remain without the referenced entity.

Example:

```text
job.worker_id
```

If a worker record is removed, the historical job remains.

### Restrict

Used for ownership.

```text
media.owner_id
```

A user should normally be deactivated rather than physically deleted.

---

# 60. Database Transaction Strategy

A media upload should approximately use:

```text
BEGIN

INSERT media

INSERT source media_asset

INSERT initial processing job

INSERT outbox event

COMMIT
```

The binary upload itself should normally happen through object storage.

The database transaction records the resulting object reference.

---

# 61. Recommended Database Schemas

Initially, all tables can remain under:

```text
public
```

As the project grows, a future structure could be:

```text
auth
media
processing
analytics
operations
audit
```

For MVP, this separation is unnecessary complexity.

---

# 62. Recommended Migration Structure

After the initial schema is validated, convert the DDL into migrations:

```text
database/
├── migrations/
│   ├── 001_extensions.sql
│   ├── 002_users.sql
│   ├── 003_media.sql
│   ├── 004_workers.sql
│   ├── 005_jobs.sql
│   ├── 006_processing.sql
│   ├── 007_analytics.sql
│   ├── 008_audit.sql
│   └── 009_indexes.sql
│
├── seeds/
│   └── development.sql
│
└── schema.sql
```

`schema.sql` should represent the complete database state.

---

# 63. Database Performance Strategy

## High-frequency queries

Indexes prioritize:

```text
jobs
media
job_events
transcripts
detections
workers
```

The most important queue query is conceptually:

```sql
SELECT *
FROM jobs
WHERE status = 'queued'
  AND available_at <= NOW()
ORDER BY priority DESC, available_at ASC
LIMIT 1;
```

The partial queue index supports this access pattern.

---

# 64. Large Detection Datasets

Detection tables may become extremely large.

For example:

```text
1 video
    ↓
100,000 frames
    ↓
5 detections/frame
    ↓
500,000 rows
```

For very large workloads, partitioning can later be introduced.

Possible strategy:

```text
detections
├── partition by media_id
```

or time-based partitioning.

Do not introduce partitioning prematurely in the MVP.

---

# 65. Transcript Query Strategy

The UI may need:

```text
Search transcript
Jump to timestamp
Show matching segment
```

The initial implementation uses PostgreSQL text search or indexed application queries.

Future optimization can introduce:

```text
GIN index
PostgreSQL full-text search
external search engine
```

depending on workload.

---

# 66. JSONB Strategy

JSONB should not become a replacement for relational design.

Good:

```json
{
  "model_version": "v2",
  "threshold": 0.75,
  "gpu": false
}
```

Bad:

```json
{
  "user_id": "...",
  "media_id": "...",
  "job_id": "..."
}
```

Frequently queried relational attributes should remain proper columns.

---

# 67. Security Considerations

Sensitive information includes:

```text
password_hash
session token hashes
IP address
audit information
```

Applications must:

* never expose password hashes
* never expose session token hashes
* restrict database access
* use encrypted connections in production
* use least-privilege database roles
* use parameterized SQL
* validate uploaded media
* avoid storing unnecessary personal data

---

# 68. Database Roles

Production should use separate roles.

Example:

```text
media_app
media_worker
media_migration
media_readonly
```

Conceptually:

```text
Node.js
    ↓
media_app

C++ / Python
    ↓
media_worker

Migration system
    ↓
media_migration

Analytics
    ↓
media_readonly
```

The application should not run migrations using its normal runtime credentials.

---

# 69. Backup Strategy

Recommended:

```text
PostgreSQL
    │
    ├── continuous WAL/archive
    ├── daily backup
    └── periodic restore test
```

Object storage must have its own backup/versioning strategy.

Database backup alone does not recover the actual media files.

---

# 70. Recovery Strategy

After PostgreSQL recovery:

```text
jobs
   │
   ▼
find queued/running jobs
   │
   ▼
reconcile Redis
   │
   ▼
resume processing
```

Because PostgreSQL is the source of truth, the queue can be rebuilt.

---

# 71. Example End-to-End Dataset

A processed video might produce:

```text
users
  1 row

media
  1 row

media_assets
  5 rows

jobs
  7 rows

job_attempts
  8 rows

job_events
  30+ rows

processing_results
  7 rows

transcripts
  1 row

transcript_segments
  120 rows

detections
  8,500 rows

scenes
  42 rows

clips
  5 rows

ai_insights
  4 rows
```

---

# 72. Final Data Architecture

```text
                         PostgreSQL
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
       Auth                Media               Operations
         │                    │                    │
     ┌───┴───┐         ┌──────┴──────┐       ┌────┴─────┐
     │       │         │             │       │          │
   Users  Sessions   Assets         Jobs   Workers    Audit
                                   │
                     ┌─────────────┼─────────────┐
                     │             │             │
                 Attempts       Events      Dependencies
                     │
                     ▼
                Processing
                     │
          ┌──────────┼──────────┐
          │          │          │
      Transcript  Detection   Scenes
          │
      Segments
          │
          └──────────┐
                     ▼
                 AI Insights
                     │
                     ▼
                   Clips
```

---

# 73. Final Storage Architecture

```text
                   Application
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
        PostgreSQL          Object Storage
             │                   │
             │                   ├── original.mp4
             │                   ├── proxy.mp4
             │                   ├── audio.wav
             │                   ├── thumbnail.jpg
             │                   ├── waveform.json
             │                   ├── clip.mp4
             │                   └── subtitles.vtt
             │
             ├── metadata
             ├── jobs
             ├── results
             ├── transcript
             ├── detections
             └── references
```

---

# 74. Definition of Done

The database implementation is considered complete when:

* [ ] PostgreSQL database can be created from a clean installation
* [ ] `schema.sql` executes without manual table creation
* [ ] all foreign keys are valid
* [ ] all constraints are enforced
* [ ] UUID primary keys work
* [ ] timestamps use `TIMESTAMPTZ`
* [ ] media metadata is separated from binary storage
* [ ] jobs support retries
* [ ] jobs support leasing
* [ ] job attempts are persisted
* [ ] job events are persisted
* [ ] job dependencies are supported
* [ ] transcripts are supported
* [ ] transcript segments are supported
* [ ] detections are supported
* [ ] scenes are supported
* [ ] clips are supported
* [ ] AI insights are supported
* [ ] workers can register
* [ ] audit events can be recorded
* [ ] outbox events can be persisted
* [ ] indexes exist for major query paths
* [ ] updated timestamps are automatically maintained
* [ ] database backup/restore has been tested

---

# 75. Next Implementation Step

After this data model, the project should move to:

```text
03 Data Model
      │
      ▼
04 API Reference
      │
      ▼
05 Roadmap & Phases
      │
      ▼
06 Development Guide
      │
      ▼
07 Security
      │
      ▼
08 Gap Analysis
      │
      ▼
09 Testing Strategy
      │
      ▼
10 Glossary
```

The next major implementation artifact after the documentation phase should be:

```text
database/
├── schema.sql
├── migrations/
├── seeds/
└── README.md
```

Then the Node.js service can connect to PostgreSQL and implement the first real API flow:

```text
POST /api/v1/media
        │
        ▼
Create media record
        │
        ▼
Create upload metadata
        │
        ▼
Create processing job
        │
        ▼
Publish queue event
```

This establishes the first complete vertical slice of the platform.
