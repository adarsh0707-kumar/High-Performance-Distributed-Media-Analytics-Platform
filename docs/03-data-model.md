# High-Performance Distributed Media Analytics Platform

## 03 — Data Model & Database Design

**Author:** Adarsh Kumar
**Version:** 1.0
**Status:** Draft / Architecture Baseline
**Database:** PostgreSQL
**Primary Purpose:** Persistent metadata, job state, processing results, audit events, and relationships

---

# 1. Document Overview

This document defines the persistent data model for the High-Performance Distributed Media Analytics Platform.

The platform processes large video and audio files through multiple services:

```text
React
  │
  ▼
Node.js API
  │
  ├──────────────► PostgreSQL
  │
  ├──────────────► Redis
  │
  └──────────────► Object Storage
                       │
              ┌────────┴────────┐
              ▼                 ▼
         C++ Workers       Python Workers
              │                 │
              └────────┬────────┘
                       ▼
                  PostgreSQL
```

PostgreSQL is responsible for **metadata and state**, not large binary media storage.

The database stores:

* users
* media records
* media assets
* processing jobs
* job dependencies
* job events
* workers
* processing results
* transcripts
* transcript segments
* detections
* scenes
* generated clips
* AI insights
* processing artifacts
* processing manifests
* audit information

Large video/audio files remain in object storage.

---

# 2. Data Modeling Goals

The database design must satisfy the following goals.

## 2.1 Consistency

Relationships between users, media, jobs, and results must remain consistent.

## 2.2 Scalability

The schema must support:

* thousands of users
* millions of media assets
* millions of processing jobs
* large numbers of transcript segments
* large numbers of detections
* continuous job-event generation

## 2.3 Query Performance

Common operations must be fast:

```text
Get user's media
Get media details
Get processing status
Get active jobs
Get transcript
Get transcript segments
Get detected faces/objects
Get generated clips
Get AI insights
Get job history
```

## 2.4 Worker Independence

C++ and Python workers must not directly manipulate arbitrary application tables.

Workers should interact through controlled job/result interfaces.

## 2.5 Fault Recovery

The database must preserve enough information to recover from:

* worker crashes
* process restarts
* queue failures
* network failures
* partial processing
* duplicate messages

## 2.6 Auditability

Important state changes must be observable.

For example:

```text
Job created
Job queued
Worker claimed job
Job started
Progress updated
Job completed
Job failed
Job retried
```

---

# 3. Database Architecture

The database is logically divided into several domains.

```text
┌─────────────────────────────────────────────┐
│                 PostgreSQL                  │
├─────────────────────────────────────────────┤
│                                             │
│ Identity Domain                             │
│ ├── users                                   │
│                                             │
│ Media Domain                                │
│ ├── media                                   │
│ ├── media_assets                            │
│ ├── artifacts                               │
│                                             │
│ Processing Domain                           │
│ ├── jobs                                    │
│ ├── job_dependencies                        │
│ ├── job_events                              │
│ ├── workers                                 │
│ ├── worker_heartbeats                       │
│                                             │
│ Analytics Domain                            │
│ ├── processing_results                      │
│ ├── transcripts                             │
│ ├── transcript_segments                     │
│ ├── detections                              │
│ ├── scenes                                  │
│ ├── clips                                   │
│ ├── ai_insights                             │
│                                             │
│ Configuration / Manifest Domain             │
│ ├── processing_manifests                    │
│                                             │
└─────────────────────────────────────────────┘
```

---

# 4. Database Naming Conventions

Tables use `snake_case`.

Examples:

```text
users
media
media_assets
processing_results
transcript_segments
job_events
```

Columns also use `snake_case`.

Examples:

```text
created_at
updated_at
media_id
worker_id
job_type
```

Primary keys use:

```text
id
```

Foreign keys use:

```text
<entity>_id
```

Examples:

```text
user_id
media_id
job_id
worker_id
```

---

# 5. Identifier Strategy

The system uses UUID identifiers.

Recommended application strategy:

```text
UUIDv7
```

stored using PostgreSQL:

```sql
uuid
```

UUIDv7 is preferred because it provides time-ordered characteristics while retaining UUID semantics.

Example:

```text
0199f1b0-7c1a-7abc-8d12-123456789abc
```

The application should generate identifiers where practical.

---

# 6. Timestamp Strategy

All persistent timestamps use:

```sql
TIMESTAMPTZ
```

Examples:

```text
created_at
updated_at
started_at
completed_at
heartbeat_at
expires_at
```

All timestamps are stored in UTC.

Application/UI layers convert UTC timestamps to the user's local timezone.

Example:

```text
Database:
2026-09-07 12:30:00 UTC

Frontend:
2026-09-07 18:00:00 IST
```

---

# 7. Core Entities

The initial database contains the following primary entities:

| Entity               | Purpose                                 |
| -------------------- | --------------------------------------- |
| users                | User accounts                           |
| media                | Logical media objects                   |
| media_assets         | Physical/object-storage representations |
| artifacts            | Generated files                         |
| jobs                 | Processing jobs                         |
| job_dependencies     | Parent/child job relationships          |
| job_events           | Job lifecycle history                   |
| workers              | Worker registrations                    |
| worker_heartbeats    | Worker health                           |
| processing_results   | Generic processing output               |
| transcripts          | Audio transcription                     |
| transcript_segments  | Timestamped transcript pieces           |
| detections           | Object/face/person detection            |
| scenes               | Scene boundaries                        |
| clips                | Generated video/audio clips             |
| ai_insights          | AI-generated analysis                   |
| processing_manifests | Processing configuration/input manifest |

---

# 8. Entity Relationship Overview

```text
                         ┌─────────────┐
                         │    users    │
                         └──────┬──────┘
                                │
                                │ 1:N
                                ▼
                         ┌─────────────┐
                         │    media    │
                         └──────┬──────┘
                                │
              ┌─────────────────┼──────────────────┐
              │                 │                  │
              ▼                 ▼                  ▼
      ┌──────────────┐  ┌──────────────┐   ┌───────────────┐
      │media_assets  │  │     jobs     │   │   artifacts   │
      └──────────────┘  └──────┬───────┘   └───────────────┘
                                │
                    ┌───────────┼────────────┐
                    │           │            │
                    ▼           ▼            ▼
              job_events    results      dependencies
                                │
               ┌────────────────┼────────────────┐
               │                │                │
               ▼                ▼                ▼
          transcripts      detections         scenes
               │
               ▼
      transcript_segments

                                │
                                ▼
                              clips

                                │
                                ▼
                          ai_insights
```

---

# 9. Users

The `users` table represents platform users.

## 9.1 Purpose

Stores authentication and account-level information.

## 9.2 Schema

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(320) NOT NULL,
    password_hash TEXT,
    display_name VARCHAR(120),
    role VARCHAR(32) NOT NULL DEFAULT 'user',

    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    email_verified BOOLEAN NOT NULL DEFAULT FALSE,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMPTZ,

    CONSTRAINT users_role_check
        CHECK (role IN ('user', 'admin'))
);
```

## 9.3 Indexes

```sql
CREATE UNIQUE INDEX users_email_unique_idx
ON users (LOWER(email));

CREATE INDEX users_active_idx
ON users (is_active);
```

---

# 10. Media

The `media` table represents the logical media object owned by a user.

A media record does not necessarily represent a single physical file.

For example:

```text
Media
 ├── original.mp4
 ├── proxy.mp4
 ├── thumbnail.jpg
 ├── audio.wav
 └── processed-output.mp4
```

All can belong to the same logical media entity.

## 10.1 Schema

```sql
CREATE TABLE media (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id),

    title VARCHAR(255),
    description TEXT,

    media_type VARCHAR(32) NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'uploaded',

    duration_ms BIGINT,
    size_bytes BIGINT,

    width INTEGER,
    height INTEGER,
    frame_rate NUMERIC(12, 6),

    audio_channels INTEGER,
    sample_rate INTEGER,

    checksum_sha256 CHAR(64),

    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,

    CONSTRAINT media_type_check
        CHECK (
            media_type IN (
                'video',
                'audio',
                'image'
            )
        ),

    CONSTRAINT media_status_check
        CHECK (
            status IN (
                'uploading',
                'uploaded',
                'processing',
                'ready',
                'failed',
                'deleted'
            )
        )
);
```

---

# 11. Media Metadata

The `metadata` JSONB column contains information that may vary between media formats.

Example:

```json
{
  "container": "mp4",
  "video_codec": "h264",
  "audio_codec": "aac",
  "bitrate": 12500000,
  "pixel_format": "yuv420p",
  "color_space": "bt709"
}
```

Stable and frequently queried fields should remain normal columns.

Do not place everything into JSONB.

---

# 12. Media Indexes

```sql
CREATE INDEX media_user_created_idx
ON media (user_id, created_at DESC);

CREATE INDEX media_status_idx
ON media (status);

CREATE INDEX media_type_idx
ON media (media_type);

CREATE INDEX media_checksum_idx
ON media (checksum_sha256);

CREATE INDEX media_metadata_gin_idx
ON media
USING GIN (metadata);
```

---

# 13. Media Assets

`media_assets` represents physical files associated with media.

Examples:

```text
original
proxy
audio
thumbnail
waveform
processed_video
subtitle
```

## 13.1 Schema

```sql
CREATE TABLE media_assets (
    id UUID PRIMARY KEY,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    asset_type VARCHAR(32) NOT NULL,

    storage_provider VARCHAR(32) NOT NULL,
    bucket_name VARCHAR(255) NOT NULL,
    object_key TEXT NOT NULL,

    mime_type VARCHAR(128),
    size_bytes BIGINT,

    checksum_sha256 CHAR(64),

    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT media_asset_type_check
        CHECK (
            asset_type IN (
                'original',
                'proxy',
                'audio',
                'thumbnail',
                'waveform',
                'processed',
                'subtitle',
                'other'
            )
        )
);
```

## 13.2 Important Design Rule

PostgreSQL stores:

```text
bucket
object key
metadata
checksum
size
```

Object storage stores:

```text
actual binary data
```

Never store multi-gigabyte video files directly in PostgreSQL for this architecture.

---

# 14. Media Asset Constraints

An asset must belong to a valid media object.

Example:

```text
media
  │
  └── media_assets
        ├── original
        ├── proxy
        └── thumbnail
```

Recommended unique constraint:

```sql
CREATE UNIQUE INDEX media_asset_object_unique_idx
ON media_assets (storage_provider, bucket_name, object_key);
```

---

# 15. Artifacts

An artifact is a generated or intermediate file produced by processing.

Examples:

```text
thumbnail.jpg
audio.wav
normalized_audio.wav
proxy.mp4
face-crops.zip
subtitle.vtt
scene-preview.mp4
clip-001.mp4
```

## 15.1 Schema

```sql
CREATE TABLE artifacts (
    id UUID PRIMARY KEY,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    job_id UUID,

    artifact_type VARCHAR(64) NOT NULL,

    storage_provider VARCHAR(32) NOT NULL,
    bucket_name VARCHAR(255) NOT NULL,
    object_key TEXT NOT NULL,

    mime_type VARCHAR(128),
    size_bytes BIGINT,

    checksum_sha256 CHAR(64),

    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

`job_id` is intentionally nullable because an artifact may survive beyond the lifetime of the job that generated it.

---

# 16. Jobs

The `jobs` table is the central processing entity.

Every asynchronous operation becomes a job.

Examples:

```text
probe_media
extract_audio
generate_thumbnail
generate_proxy
decode_video
transcribe_audio
detect_faces
detect_objects
detect_scenes
generate_clip
generate_summary
sentiment_analysis
```

---

# 17. Job Schema

```sql
CREATE TABLE jobs (
    id UUID PRIMARY KEY,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    parent_job_id UUID REFERENCES jobs(id),

    job_type VARCHAR(64) NOT NULL,

    status VARCHAR(32) NOT NULL DEFAULT 'pending',

    priority INTEGER NOT NULL DEFAULT 100,

    attempt_count INTEGER NOT NULL DEFAULT 0,
    max_attempts INTEGER NOT NULL DEFAULT 3,

    progress NUMERIC(5, 2) NOT NULL DEFAULT 0,

    worker_id UUID,

    input JSONB NOT NULL DEFAULT '{}'::jsonb,
    output JSONB NOT NULL DEFAULT '{}'::jsonb,
    error JSONB,

    idempotency_key VARCHAR(255),

    queued_at TIMESTAMPTZ,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT jobs_status_check
        CHECK (
            status IN (
                'pending',
                'queued',
                'running',
                'completed',
                'failed',
                'cancelled',
                'retrying'
            )
        ),

    CONSTRAINT jobs_progress_check
        CHECK (
            progress >= 0 AND progress <= 100
        ),

    CONSTRAINT jobs_attempt_check
        CHECK (
            attempt_count >= 0
        )
);
```

---

# 18. Job Types

Initial job types:

| Job Type             | Worker |
| -------------------- | ------ |
| probe_media          | C++    |
| decode_video         | C++    |
| extract_audio        | C++    |
| downmix_audio        | C++    |
| generate_thumbnail   | C++    |
| generate_proxy       | C++    |
| detect_scenes        | C++    |
| transcribe_audio     | Python |
| speech_analysis      | Python |
| face_detection       | Python |
| object_detection     | Python |
| sentiment_analysis   | Python |
| summarize_transcript | Python |
| generate_clip        | C++    |
| final_analysis       | Python |

---

# 19. Job State Machine

```text
                 ┌─────────────┐
                 │   pending   │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │   queued    │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │   running   │
                 └───┬─────┬───┘
                     │     │
              success│     │failure
                     │     │
                     ▼     ▼
              ┌─────────┐ ┌──────────┐
              │completed│ │ retrying │
              └─────────┘ └────┬─────┘
                                │
                                ▼
                              queued

running ───────────────► cancelled
running ───────────────► failed
```

---

# 20. Job Dependencies

Some processing jobs cannot begin until another job completes.

Example:

```text
probe_media
     │
     ├──► generate_thumbnail
     │
     ├──► extract_audio
     │          │
     │          ▼
     │     transcribe_audio
     │          │
     │          ▼
     │     summarize_transcript
     │
     └──► detect_scenes
```

---

# 21. Job Dependencies Table

```sql
CREATE TABLE job_dependencies (
    job_id UUID NOT NULL REFERENCES jobs(id)
        ON DELETE CASCADE,

    depends_on_job_id UUID NOT NULL REFERENCES jobs(id)
        ON DELETE CASCADE,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (job_id, depends_on_job_id),

    CONSTRAINT job_dependency_self_check
        CHECK (job_id <> depends_on_job_id)
);
```

---

# 22. Job Dependency Rules

A job can execute only when all required dependencies are completed.

Example:

```text
transcribe_audio
```

depends on:

```text
extract_audio
```

The queue/orchestrator checks:

```text
Are all dependencies completed?
        │
        ├── NO → remain pending
        │
        └── YES → enqueue
```

---

# 23. Job Events

`job_events` provides an append-only history of job state changes.

## 23.1 Schema

```sql
CREATE TABLE job_events (
    id UUID PRIMARY KEY,

    job_id UUID NOT NULL REFERENCES jobs(id)
        ON DELETE CASCADE,

    event_type VARCHAR(64) NOT NULL,

    previous_status VARCHAR(32),
    new_status VARCHAR(32),

    progress NUMERIC(5, 2),

    worker_id UUID,

    message TEXT,

    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 24. Job Event Examples

```text
JOB_CREATED
JOB_QUEUED
JOB_CLAIMED
JOB_STARTED
JOB_PROGRESS
JOB_COMPLETED
JOB_FAILED
JOB_RETRYING
JOB_CANCELLED
JOB_TIMEOUT
JOB_RECOVERED
```

Example event:

```json
{
  "event_type": "JOB_PROGRESS",
  "progress": 63.5,
  "message": "Decoding video frames"
}
```

---

# 25. Job Event Design

Job events should be treated as append-only.

Do not update old events.

Correct:

```text
event 1 → created
event 2 → queued
event 3 → started
event 4 → progress
event 5 → completed
```

Incorrect:

```text
UPDATE old event
```

This provides an audit trail.

---

# 26. Workers

Workers are registered in the database.

## 26.1 Schema

```sql
CREATE TABLE workers (
    id UUID PRIMARY KEY,

    worker_type VARCHAR(32) NOT NULL,

    hostname VARCHAR(255),
    process_id INTEGER,

    version VARCHAR(64),

    status VARCHAR(32) NOT NULL DEFAULT 'online',

    capabilities JSONB NOT NULL DEFAULT '[]'::jsonb,

    last_heartbeat_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT worker_type_check
        CHECK (
            worker_type IN (
                'cpp',
                'python'
            )
        ),

    CONSTRAINT worker_status_check
        CHECK (
            status IN (
                'online',
                'busy',
                'offline',
                'draining'
            )
        )
);
```

---

# 27. Worker Capabilities

Example:

```json
[
  "decode_video",
  "extract_audio",
  "generate_thumbnail",
  "generate_proxy",
  "detect_scenes"
]
```

Python worker:

```json
[
  "transcribe_audio",
  "face_detection",
  "object_detection",
  "summarization"
]
```

---

# 28. Worker Heartbeats

Worker health requires a dedicated heartbeat history.

```sql
CREATE TABLE worker_heartbeats (
    id UUID PRIMARY KEY,

    worker_id UUID NOT NULL REFERENCES workers(id)
        ON DELETE CASCADE,

    cpu_percent NUMERIC(5, 2),
    memory_percent NUMERIC(5, 2),

    active_jobs INTEGER NOT NULL DEFAULT 0,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

The platform can determine:

```text
Healthy
    ↓
heartbeat every N seconds
    ↓
No heartbeat
    ↓
timeout threshold
    ↓
worker considered unavailable
```

---

# 29. Processing Results

`processing_results` provides a generic connection between jobs and generated analytical results.

```sql
CREATE TABLE processing_results (
    id UUID PRIMARY KEY,

    job_id UUID NOT NULL REFERENCES jobs(id)
        ON DELETE CASCADE,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    result_type VARCHAR(64) NOT NULL,

    schema_version VARCHAR(32) NOT NULL DEFAULT '1.0',

    result JSONB NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Example:

```json
{
  "codec": "h264",
  "duration_ms": 125000,
  "frame_count": 3750
}
```

---

# 30. Why Generic Results Exist

Not every processing result deserves its own table.

For example:

```text
media_probe
codec_info
processing_statistics
model_metadata
quality_metrics
```

can be stored as JSONB.

However, highly queryable structures such as:

```text
transcript_segments
detections
scenes
clips
```

should have normalized tables.

---

# 31. Transcripts

A transcript belongs to a media object.

```sql
CREATE TABLE transcripts (
    id UUID PRIMARY KEY,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    job_id UUID REFERENCES jobs(id),

    language VARCHAR(16),
    model_name VARCHAR(128),

    confidence NUMERIC(5, 4),

    full_text TEXT,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 32. Transcript Segments

A transcript is divided into timestamped segments.

Example:

```text
00:00:01.200 → 00:00:04.800
"Welcome to the media analytics platform."

00:00:05.100 → 00:00:08.600
"Today we will process this video."
```

## Schema

```sql
CREATE TABLE transcript_segments (
    id UUID PRIMARY KEY,

    transcript_id UUID NOT NULL REFERENCES transcripts(id)
        ON DELETE CASCADE,

    segment_index INTEGER NOT NULL,

    start_ms BIGINT NOT NULL,
    end_ms BIGINT NOT NULL,

    text TEXT NOT NULL,

    confidence NUMERIC(5, 4),

    speaker_id VARCHAR(128),

    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT transcript_segment_time_check
        CHECK (end_ms >= start_ms),

    CONSTRAINT transcript_segment_confidence_check
        CHECK (
            confidence IS NULL
            OR (
                confidence >= 0
                AND confidence <= 1
            )
        )
);
```

---

# 33. Transcript Indexes

```sql
CREATE UNIQUE INDEX transcript_segment_order_idx
ON transcript_segments (
    transcript_id,
    segment_index
);

CREATE INDEX transcript_segment_time_idx
ON transcript_segments (
    transcript_id,
    start_ms,
    end_ms
);
```

This allows timeline queries such as:

```sql
SELECT *
FROM transcript_segments
WHERE transcript_id = $1
AND start_ms <= $2
AND end_ms >= $2;
```

---

# 34. Detections

The detection table stores objects, faces, people, and other visual detections.

Example:

```text
Frame 100
 ├── person
 ├── car
 └── face

Frame 101
 ├── person
 ├── car
 └── face
```

---

# 35. Detection Schema

```sql
CREATE TABLE detections (
    id UUID PRIMARY KEY,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    job_id UUID REFERENCES jobs(id),

    detection_type VARCHAR(64) NOT NULL,
    label VARCHAR(128) NOT NULL,

    confidence NUMERIC(5, 4) NOT NULL,

    start_ms BIGINT,
    end_ms BIGINT,

    frame_number BIGINT,

    x NUMERIC(12, 6),
    y NUMERIC(12, 6),
    width NUMERIC(12, 6),
    height NUMERIC(12, 6),

    track_id VARCHAR(128),

    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT detection_confidence_check
        CHECK (
            confidence >= 0 AND confidence <= 1
        )
);
```

---

# 36. Detection Types

Examples:

```text
face
person
object
vehicle
logo
gesture
scene_object
```

Example record:

```json
{
  "detection_type": "object",
  "label": "car",
  "confidence": 0.9732,
  "start_ms": 1200,
  "end_ms": 5400,
  "x": 0.32,
  "y": 0.41,
  "width": 0.20,
  "height": 0.18
}
```

Coordinates are normalized:

```text
0.0 → left/top
1.0 → right/bottom
```

---

# 37. Detection Indexes

```sql
CREATE INDEX detections_media_time_idx
ON detections (
    media_id,
    start_ms,
    end_ms
);

CREATE INDEX detections_type_idx
ON detections (
    media_id,
    detection_type
);

CREATE INDEX detections_label_idx
ON detections (
    media_id,
    label
);

CREATE INDEX detections_track_idx
ON detections (
    media_id,
    track_id
);
```

For extremely large detection datasets, partitioning can be considered later.

---

# 38. Scenes

A scene represents a continuous logical section of media.

Example:

```text
Scene 1
00:00 → 00:18

Scene 2
00:18 → 00:42

Scene 3
00:42 → 01:12
```

## Schema

```sql
CREATE TABLE scenes (
    id UUID PRIMARY KEY,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    job_id UUID REFERENCES jobs(id),

    scene_index INTEGER NOT NULL,

    start_ms BIGINT NOT NULL,
    end_ms BIGINT NOT NULL,

    confidence NUMERIC(5, 4),

    label VARCHAR(128),

    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT scene_time_check
        CHECK (end_ms >= start_ms)
);
```

---

# 39. Scene Index

```sql
CREATE UNIQUE INDEX scenes_media_order_idx
ON scenes (
    media_id,
    scene_index
);

CREATE INDEX scenes_media_time_idx
ON scenes (
    media_id,
    start_ms,
    end_ms
);
```

---

# 40. Clips

Clips are generated media segments.

A clip can be created automatically or requested by a user.

Examples:

```text
Highlight clip
Face appearance clip
Scene clip
AI-generated short
User-selected clip
```

---

# 41. Clip Schema

```sql
CREATE TABLE clips (
    id UUID PRIMARY KEY,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    job_id UUID REFERENCES jobs(id),

    artifact_id UUID REFERENCES artifacts(id),

    name VARCHAR(255),

    start_ms BIGINT NOT NULL,
    end_ms BIGINT NOT NULL,

    clip_type VARCHAR(64) NOT NULL,

    status VARCHAR(32) NOT NULL DEFAULT 'processing',

    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT clip_time_check
        CHECK (end_ms > start_ms),

    CONSTRAINT clip_status_check
        CHECK (
            status IN (
                'processing',
                'ready',
                'failed',
                'deleted'
            )
        )
);
```

---

# 42. AI Insights

AI insights contain higher-level interpretations.

Examples:

```text
summary
sentiment
keywords
topics
entities
content classification
safety classification
highlights
```

---

# 43. AI Insight Schema

```sql
CREATE TABLE ai_insights (
    id UUID PRIMARY KEY,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    job_id UUID REFERENCES jobs(id),

    insight_type VARCHAR(64) NOT NULL,

    model_name VARCHAR(128),
    model_version VARCHAR(64),

    confidence NUMERIC(5, 4),

    content JSONB NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Example:

```json
{
  "insight_type": "summary",
  "content": {
    "summary": "The video demonstrates the media processing pipeline.",
    "keywords": [
      "FFmpeg",
      "C++",
      "Python",
      "AI"
    ]
  }
}
```

---

# 44. Processing Manifests

A processing manifest describes exactly what a worker should process.

Example:

```json
{
  "media_id": "0199...",
  "input_asset": {
    "bucket": "media",
    "key": "original/video.mp4"
  },
  "operations": [
    "decode",
    "extract_audio"
  ],
  "parameters": {
    "sample_rate": 16000,
    "audio_channels": 1
  }
}
```

---

# 45. Processing Manifest Schema

```sql
CREATE TABLE processing_manifests (
    id UUID PRIMARY KEY,

    media_id UUID NOT NULL REFERENCES media(id)
        ON DELETE CASCADE,

    job_id UUID REFERENCES jobs(id),

    schema_version VARCHAR(32) NOT NULL DEFAULT '1.0',

    manifest JSONB NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 46. JSONB Strategy

JSONB is intentionally used for flexible metadata.

Good JSONB candidates:

```text
model parameters
codec-specific metadata
worker capabilities
AI output
optional processing configuration
result metadata
```

Bad JSONB candidates:

```text
user_id
media_id
job_id
status
created_at
start_ms
end_ms
worker_id
```

Frequently filtered or joined fields should remain relational columns.

---

# 47. Foreign-Key Relationship Summary

```text
users
  │
  └── media
        │
        ├── media_assets
        │
        ├── artifacts
        │
        ├── jobs
        │     │
        │     ├── job_dependencies
        │     ├── job_events
        │     ├── processing_results
        │     └── processing_manifests
        │
        ├── transcripts
        │     └── transcript_segments
        │
        ├── detections
        │
        ├── scenes
        │
        ├── clips
        │
        └── ai_insights
```

---

# 48. Cascading Delete Strategy

Media-owned analytical records can be deleted when a media object is permanently deleted.

For example:

```text
media
 ├── transcripts
 ├── scenes
 ├── detections
 ├── clips
 └── ai_insights
```

These use:

```sql
ON DELETE CASCADE
```

However, important audit records should generally have independent retention policies.

---

# 49. Soft Delete

Media supports:

```text
deleted_at
```

rather than immediately deleting the record.

Example:

```sql
UPDATE media
SET deleted_at = NOW(),
    status = 'deleted'
WHERE id = $1;
```

Normal application queries should use:

```sql
WHERE deleted_at IS NULL
```

Physical deletion can happen later through a retention worker.

---

# 50. Recommended Partial Index

```sql
CREATE INDEX media_active_user_idx
ON media (
    user_id,
    created_at DESC
)
WHERE deleted_at IS NULL;
```

This improves normal media-library queries.

---

# 51. Idempotency

Processing operations must be safe against duplicate messages.

Example:

```text
Job:
transcribe_audio

Idempotency key:
media-id + operation + configuration-version
```

Database:

```sql
CREATE UNIQUE INDEX jobs_idempotency_idx
ON jobs (
    media_id,
    idempotency_key
)
WHERE idempotency_key IS NOT NULL;
```

This prevents accidental duplicate jobs.

---

# 52. Example Idempotency Key

```text
media:0199abc...
operation:transcribe_audio
version:v1
```

Could be represented as:

```text
0199abc...:transcribe_audio:v1
```

---

# 53. Job Concurrency

Workers may attempt to claim the same job.

The system must prevent two workers from processing the same job simultaneously.

A common PostgreSQL strategy is:

```sql
SELECT id
FROM jobs
WHERE status = 'queued'
ORDER BY priority ASC, created_at ASC
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

Then:

```text
transaction
    ↓
lock job
    ↓
assign worker
    ↓
status = running
    ↓
commit
```

This is especially useful when multiple workers consume jobs from a database-backed coordination path.

Redis remains the primary queue in the target architecture, but PostgreSQL still needs correct concurrency semantics.

---

# 54. Worker Job Ownership

When a worker claims a job:

```text
jobs.worker_id = worker.id
jobs.status = running
jobs.started_at = NOW()
```

Example:

```sql
UPDATE jobs
SET
    worker_id = $worker_id,
    status = 'running',
    started_at = COALESCE(started_at, NOW()),
    attempt_count = attempt_count + 1,
    updated_at = NOW()
WHERE id = $job_id
AND status = 'queued';
```

The update must verify that the job is still claimable.

---

# 55. Worker Failure Recovery

Suppose:

```text
C++ Worker
   │
   └── processing job
          │
          X crash
```

The system detects:

```text
heartbeat timeout
```

Then:

```text
worker → offline
job → recovery candidate
```

The recovery process checks:

```text
attempt_count < max_attempts
```

If true:

```text
job → retrying → queued
```

Otherwise:

```text
job → failed
```

---

# 56. Transaction Boundaries

Transactions should be short.

Good:

```text
BEGIN
create job
create job event
COMMIT
```

Bad:

```text
BEGIN
start video processing
decode 2 GB video
run AI
upload output
COMMIT
```

Never hold a PostgreSQL transaction during long media processing.

---

# 57. Processing Transaction Pattern

Correct architecture:

```text
Transaction 1
    Create job
    Commit

Redis
    Queue job

Worker
    Process media

Transaction 2
    Store processing result
    Update job status
    Create completion event
    Commit
```

---

# 58. Result Persistence

A worker should not keep PostgreSQL transactions open while processing.

Instead:

```text
Worker starts
   │
   ▼
Download/read input
   │
   ▼
Process
   │
   ▼
Upload artifact
   │
   ▼
Persist result metadata
   │
   ▼
Mark job completed
```

---

# 59. Data Lifecycle

A media object passes through several stages.

```text
Upload
  │
  ▼
Media created
  │
  ▼
Original stored
  │
  ▼
Probe
  │
  ▼
Processing
  │
  ├── transcript
  ├── detections
  ├── scenes
  ├── clips
  └── AI insights
  │
  ▼
Ready
  │
  ▼
Archived
  │
  ▼
Deleted
```

---

# 60. Retention Strategy

Different data classes may have different retention policies.

| Data                | Suggested Retention      |
| ------------------- | ------------------------ |
| Original media      | User-defined             |
| Generated proxies   | Temporary / configurable |
| Thumbnails          | Long-lived               |
| Job records         | Long-lived               |
| Job events          | Configurable             |
| Worker heartbeats   | Short-lived              |
| AI results          | Long-lived               |
| Temporary artifacts | Short-lived              |
| Audit information   | Long-lived               |

Actual production retention periods should be configurable.

---

# 61. Worker Heartbeat Retention

Heartbeat records can grow extremely quickly.

Example:

```text
100 workers
heartbeat every 10 seconds
```

That produces:

```text
600 heartbeats/minute
36,000/hour
864,000/day
```

Therefore heartbeat history should be periodically aggregated or deleted.

For example:

```text
Raw heartbeat:
retain 24 hours

Aggregated worker metrics:
retain 30–90 days
```

---

# 62. Large-Table Considerations

Potentially large tables include:

```text
job_events
worker_heartbeats
transcript_segments
detections
```

These should be monitored.

Possible future optimization:

```text
PostgreSQL partitioning
```

For example:

```text
detections
 ├── partition_2026_01
 ├── partition_2026_02
 ├── partition_2026_03
 └── ...
```

Do not introduce partitioning prematurely.

Start with correct indexes and measure first.

---

# 63. Database Indexing Strategy

Primary indexes:

```text
users.email

media.user_id + created_at
media.status

media_assets.media_id

jobs.media_id
jobs.status
jobs.worker_id
jobs.priority + created_at

job_events.job_id + created_at

transcripts.media_id

transcript_segments.transcript_id + start_ms

detections.media_id + start_ms

scenes.media_id + start_ms

clips.media_id

ai_insights.media_id
```

---

# 64. Job Queue Query Index

For queue-like database queries:

```sql
CREATE INDEX jobs_queue_idx
ON jobs (
    priority,
    created_at
)
WHERE status = 'queued';
```

This keeps the active queue index smaller.

---

# 65. Job Status Index

```sql
CREATE INDEX jobs_status_created_idx
ON jobs (
    status,
    created_at DESC
);
```

Useful for:

```text
dashboard
worker monitoring
admin operations
recovery
```

---

# 66. Job Worker Index

```sql
CREATE INDEX jobs_worker_status_idx
ON jobs (
    worker_id,
    status
);
```

Useful for:

```text
What is worker X currently processing?
```

---

# 67. Job Events Index

```sql
CREATE INDEX job_events_job_created_idx
ON job_events (
    job_id,
    created_at DESC
);
```

This supports:

```text
GET /jobs/:id/events
```

efficiently.

---

# 68. Data Access Rules

The Node.js API is the primary database client.

```text
React
  │
  ▼
Node.js
  │
  ▼
PostgreSQL
```

Workers should not receive unrestricted database credentials.

Preferred:

```text
Worker
  │
  ▼
Worker service/API
  │
  ▼
Database
```

or a tightly restricted worker database role.

---

# 69. Database Roles

Recommended roles:

```text
app_user
worker_user
migration_user
readonly_user
```

Example permissions:

```text
migration_user
    CREATE / ALTER / DROP

app_user
    SELECT / INSERT / UPDATE / DELETE

worker_user
    restricted SELECT / INSERT / UPDATE

readonly_user
    SELECT
```

---

# 70. Security Boundaries

Never store:

```text
raw passwords
```

Store:

```text
password_hash
```

using a modern password hashing algorithm through the authentication subsystem.

Object storage credentials should not be stored in media records.

Instead:

```text
environment variables
secret manager
service identity
```

---

# 71. Row Ownership

Every media object belongs to a user.

```text
media.user_id
```

API authorization must enforce:

```text
request.user_id == media.user_id
```

or appropriate administrative authorization.

Do not rely only on frontend filtering.

---

# 72. Example User Query

```sql
SELECT
    id,
    title,
    media_type,
    status,
    duration_ms,
    created_at
FROM media
WHERE user_id = $1
AND deleted_at IS NULL
ORDER BY created_at DESC
LIMIT $2
OFFSET $3;
```

---

# 73. Example Media Detail Query

```sql
SELECT
    m.*,
    ma.asset_type,
    ma.bucket_name,
    ma.object_key
FROM media m
LEFT JOIN media_assets ma
    ON ma.media_id = m.id
WHERE m.id = $1
AND m.deleted_at IS NULL;
```

---

# 74. Example Job Status Query

```sql
SELECT
    id,
    job_type,
    status,
    progress,
    attempt_count,
    started_at,
    completed_at,
    error
FROM jobs
WHERE media_id = $1
ORDER BY created_at ASC;
```

---

# 75. Example Transcript Query

```sql
SELECT
    ts.start_ms,
    ts.end_ms,
    ts.text,
    ts.confidence,
    ts.speaker_id
FROM transcript_segments ts
JOIN transcripts t
    ON t.id = ts.transcript_id
WHERE t.media_id = $1
ORDER BY ts.segment_index;
```

---

# 76. Example Timeline Query

The frontend may request all events for a specific timeline position.

```sql
SELECT *
FROM detections
WHERE media_id = $1
AND start_ms <= $2
AND end_ms >= $2
ORDER BY confidence DESC;
```

---

# 77. Data Consistency Rules

The following invariants must always hold.

### Rule 1

Every media object belongs to a valid user.

```text
media.user_id → users.id
```

### Rule 2

Every job belongs to a valid media object.

```text
jobs.media_id → media.id
```

### Rule 3

Every transcript belongs to media.

```text
transcripts.media_id → media.id
```

### Rule 4

Every transcript segment belongs to a transcript.

### Rule 5

Every detection belongs to media.

### Rule 6

Every generated clip belongs to media.

### Rule 7

Completed jobs must have completion information.

### Rule 8

Running jobs must have a worker owner unless explicitly designed for recovery.

---

# 78. Job Completion Rule

Application-level validation should ensure:

```text
status = completed
```

has:

```text
completed_at IS NOT NULL
```

Likewise:

```text
status = running
```

should normally have:

```text
started_at IS NOT NULL
```

These can be enforced partly through application logic and, where appropriate, database constraints.

---

# 79. Error Model

The `jobs.error` JSONB field stores structured failure information.

Example:

```json
{
  "code": "FFMPEG_DECODE_ERROR",
  "message": "Unable to decode input stream",
  "retryable": false,
  "component": "cpp-media-worker",
  "details": {
    "stream_index": 0,
    "codec": "h264"
  }
}
```

Do not store sensitive credentials or secrets in error objects.

---

# 80. Error Categories

Recommended categories:

```text
INVALID_INPUT
MEDIA_CORRUPTED
UNSUPPORTED_FORMAT
FFMPEG_ERROR
WORKER_ERROR
MODEL_ERROR
TIMEOUT
STORAGE_ERROR
NETWORK_ERROR
DATABASE_ERROR
INTERNAL_ERROR
```

---

# 81. Retry Policy

Retryable examples:

```text
NETWORK_ERROR
TEMPORARY_STORAGE_ERROR
WORKER_CRASH
TIMEOUT
TRANSIENT_DATABASE_ERROR
```

Non-retryable examples:

```text
INVALID_INPUT
MEDIA_CORRUPTED
UNSUPPORTED_FORMAT
INVALID_CONFIGURATION
```

Example:

```text
attempt 1 → failed
attempt 2 → failed
attempt 3 → failed
            ↓
          dead/final failed
```

---

# 82. Processing Result Versioning

Processing output can change when models or algorithms change.

Therefore:

```text
schema_version
model_version
processor_version
```

should be recorded where applicable.

Example:

```json
{
  "processor": "face-detector",
  "processor_version": "1.4.0",
  "model": "yolovX",
  "model_version": "2026.02"
}
```

This allows results to be reproduced or compared.

---

# 83. Reprocessing

The system should support reprocessing the same media.

Example:

```text
video.mp4
   │
   ├── processing run #1
   │       model v1
   │
   └── processing run #2
           model v2
```

Do not overwrite historical analytical results blindly.

A future enhancement can introduce:

```text
processing_runs
```

to group all jobs/results belonging to one execution.

---

# 84. Recommended Future Entity: Processing Runs

For the initial MVP, jobs can directly reference media.

As the platform becomes more advanced, add:

```sql
processing_runs
```

with:

```text
id
media_id
pipeline_version
configuration
status
started_at
completed_at
```

Then:

```text
processing_run
       │
       ├── jobs
       ├── artifacts
       ├── results
       └── AI outputs
```

This is recommended for the production evolution of the platform.

---

# 85. Current MVP vs Future Schema

## MVP

```text
users
media
media_assets
artifacts
jobs
job_dependencies
job_events
workers
worker_heartbeats
processing_results
transcripts
transcript_segments
detections
scenes
clips
ai_insights
processing_manifests
```

## Future

```text
processing_runs
projects
folders
media_versions
model_registry
pipeline_definitions
annotations
search_indexes
usage_records
billing_records
webhook_deliveries
```

---

# 86. Full Initial DDL Order

Migrations should create tables in dependency order.

Recommended:

```text
001_extensions
002_users
003_media
004_media_assets
005_workers
006_jobs
007_job_dependencies
008_job_events
009_worker_heartbeats
010_artifacts
011_processing_results
012_processing_manifests
013_transcripts
014_transcript_segments
015_detections
016_scenes
017_clips
018_ai_insights
019_indexes
020_constraints
```

---

# 87. Migration Strategy

Database schema changes must be version controlled.

Example:

```text
database/
└── migrations/
    ├── 001_create_users.sql
    ├── 002_create_media.sql
    ├── 003_create_media_assets.sql
    ├── 004_create_jobs.sql
    ├── 005_create_job_events.sql
    └── ...
```

Never manually modify production schema without a migration.

---

# 88. Migration Principles

Every migration should be:

```text
versioned
repeatable in a controlled environment
reviewable
tested
backward-aware where possible
```

Avoid destructive migrations without a migration plan.

For example, instead of immediately:

```sql
DROP COLUMN old_field;
```

use:

```text
1. Add new field
2. Deploy code using new field
3. Migrate data
4. Remove old field later
```

---

# 89. Seed Data

Development environments should include seed data.

Example:

```text
admin@example.local
user@example.local
```

Example media:

```text
demo-video.mp4
demo-audio.wav
```

Example jobs:

```text
probe_media
extract_audio
transcribe_audio
detect_scenes
```

Production must never use development credentials.

---

# 90. Example Job Record

```json
{
  "id": "0199f1b0-7c1a-7abc-8d12-123456789abc",
  "media_id": "0199f1a1-1234-7abc-8d12-123456789abc",
  "job_type": "transcribe_audio",
  "status": "running",
  "priority": 100,
  "attempt_count": 1,
  "max_attempts": 3,
  "progress": 47.5,
  "input": {
    "language": "en",
    "model": "whisper"
  }
}
```

---

# 91. Example Processing Result

```json
{
  "result_type": "media_probe",
  "schema_version": "1.0",
  "result": {
    "duration_ms": 125000,
    "width": 1920,
    "height": 1080,
    "frame_rate": 30,
    "video_codec": "h264",
    "audio_codec": "aac"
  }
}
```

---

# 92. Example Transcript

```json
{
  "language": "en",
  "model_name": "whisper",
  "confidence": 0.94,
  "full_text": "Welcome to the media analytics platform."
}
```

Segment:

```json
{
  "segment_index": 0,
  "start_ms": 1200,
  "end_ms": 4800,
  "text": "Welcome to the media analytics platform.",
  "confidence": 0.97
}
```

---

# 93. Example Detection

```json
{
  "detection_type": "person",
  "label": "person",
  "confidence": 0.982,
  "start_ms": 1200,
  "end_ms": 5500,
  "x": 0.32,
  "y": 0.18,
  "width": 0.24,
  "height": 0.61,
  "track_id": "person-001"
}
```

---

# 94. Example AI Insight

```json
{
  "insight_type": "summary",
  "model_name": "llm-summary",
  "model_version": "1.0",
  "confidence": 0.91,
  "content": {
    "summary": "A technical demonstration of distributed media processing.",
    "topics": [
      "video processing",
      "FFmpeg",
      "AI",
      "distributed systems"
    ],
    "keywords": [
      "C++",
      "Python",
      "Redis",
      "PostgreSQL"
    ]
  }
}
```

---

# 95. Database-to-Object-Storage Relationship

The database never needs to know the contents of the object.

Instead:

```text
PostgreSQL
┌─────────────────────────┐
│ media_assets            │
│                         │
│ bucket = media          │
│ object_key = abc/x.mp4  │
│ checksum = ...          │
└────────────┬────────────┘
             │
             ▼
Object Storage
┌─────────────────────────┐
│ media/                  │
│ └── abc/                │
│     └── x.mp4           │
└─────────────────────────┘
```

---

# 96. Artifact Lifecycle

```text
Job starts
   │
   ▼
Worker generates artifact
   │
   ▼
Upload artifact
   │
   ▼
Create artifact DB record
   │
   ▼
Reference artifact from result/clip
   │
   ▼
Available to frontend
```

---

# 97. Database Backup

PostgreSQL backups must cover:

```text
users
media metadata
jobs
processing results
transcripts
detections
scenes
clips metadata
AI insights
```

Object storage requires its own backup/versioning strategy.

A database backup does **not** replace an object-storage backup.

---

# 98. Recovery Model

If PostgreSQL is restored:

```text
database metadata restored
```

Object storage must also be available.

Therefore disaster recovery consists of:

```text
PostgreSQL backup
+
Object storage durability/backup
+
Redis recovery strategy
```

Redis should not be treated as the permanent source of truth.

PostgreSQL remains the authoritative persistent state.

---

# 99. Redis vs PostgreSQL Responsibilities

## Redis

Used for:

```text
job dispatch
temporary queue state
real-time events
short-lived coordination
rate limiting
caching
```

## PostgreSQL

Used for:

```text
persistent job state
media metadata
results
audit events
users
analytics
relationships
```

Therefore:

```text
Redis = fast coordination

PostgreSQL = durable truth
```

---

# 100. Data Flow Example

A user uploads:

```text
video.mp4
```

Flow:

```text
1. User
   │
   ▼
2. React
   │
   ▼
3. Node.js
   │
   ├── creates media row
   │
   └── obtains storage upload
   │
   ▼
4. Object Storage
   │
   ▼
5. media_assets row
   │
   ▼
6. jobs row
   │
   ▼
7. Redis
   │
   ▼
8. C++ Worker
   │
   ├── probe
   ├── extract audio
   ├── thumbnail
   └── scene detection
   │
   ▼
9. Results
   │
   ▼
10. PostgreSQL
   │
   ▼
11. Python Worker
   │
   ├── transcription
   ├── object/face detection
   └── AI analysis
   │
   ▼
12. PostgreSQL
   │
   ▼
13. React Media Studio
```

---

# 101. API Mapping

Database entities map to API resources.

| Database     | API                            |
| ------------ | ------------------------------ |
| users        | `/api/v1/users`                |
| media        | `/api/v1/media`                |
| media_assets | `/api/v1/media/:id/assets`     |
| jobs         | `/api/v1/jobs`                 |
| job_events   | `/api/v1/jobs/:id/events`      |
| transcripts  | `/api/v1/media/:id/transcript` |
| detections   | `/api/v1/media/:id/detections` |
| scenes       | `/api/v1/media/:id/scenes`     |
| clips        | `/api/v1/media/:id/clips`      |
| ai_insights  | `/api/v1/media/:id/insights`   |

---

# 102. Frontend Data Model

React should not mirror every database table directly.

Instead, API DTOs should expose frontend-oriented models.

Example:

```typescript
interface MediaSummary {
    id: string;
    title: string;
    mediaType: "video" | "audio" | "image";
    status: string;
    durationMs: number | null;
    thumbnailUrl?: string;
    createdAt: string;
}
```

This prevents the frontend from becoming coupled to PostgreSQL schema details.

---

# 103. Node.js Data Access Layer

Recommended structure:

```text
gateway-node/
└── src/
    ├── modules/
    │   ├── users/
    │   │   ├── user.repository.ts
    │   │   ├── user.service.ts
    │   │   └── user.controller.ts
    │   │
    │   ├── media/
    │   │   ├── media.repository.ts
    │   │   ├── media.service.ts
    │   │   └── media.controller.ts
    │   │
    │   └── jobs/
    │       ├── job.repository.ts
    │       ├── job.service.ts
    │       └── job.controller.ts
    │
    └── database/
        ├── pool.ts
        └── migrations/
```

---

# 104. Repository Responsibilities

Repositories handle database operations.

Example:

```text
MediaRepository
    create()
    findById()
    findByUser()
    updateStatus()
    softDelete()
```

Job repository:

```text
JobRepository
    create()
    findById()
    updateStatus()
    claim()
    retry()
    fail()
```

Services contain business logic.

Controllers handle HTTP.

---

# 105. Business Logic Boundary

Do not put business logic directly inside SQL.

Example:

Bad:

```text
Controller
   ↓
SQL
   ↓
Response
```

Preferred:

```text
Controller
   ↓
Service
   ↓
Repository
   ↓
PostgreSQL
```

---

# 106. Observability

Database operations should be correlated with:

```text
request_id
job_id
media_id
worker_id
user_id
```

Example log:

```json
{
  "request_id": "req-123",
  "job_id": "job-456",
  "media_id": "media-789",
  "worker_id": "worker-001",
  "event": "job_completed"
}
```

This allows tracing:

```text
HTTP request
    ↓
job
    ↓
worker
    ↓
database result
```

---

# 107. Performance Considerations

Avoid:

```text
SELECT *
```

for large tables.

Prefer:

```sql
SELECT id, job_type, status, progress
FROM jobs
WHERE media_id = $1;
```

Use:

```text
pagination
indexes
keyset pagination
batch inserts
prepared statements
connection pooling
```

---

# 108. Pagination

For large media libraries, prefer cursor/keyset pagination.

Instead of:

```sql
OFFSET 100000
```

use:

```sql
WHERE created_at < $cursor
ORDER BY created_at DESC
LIMIT 50;
```

This remains efficient as datasets grow.

---

# 109. Batch Inserts

Detection workers may generate thousands of records.

Avoid:

```text
INSERT
INSERT
INSERT
INSERT
...
```

Use:

```text
batch insert
```

or PostgreSQL `COPY` where appropriate.

Example:

```text
10,000 detections
        ↓
batch persistence
        ↓
PostgreSQL
```

---

# 110. Connection Pooling

Node.js should use a PostgreSQL connection pool.

Conceptually:

```text
Node.js
   │
   ▼
Connection Pool
 ┌───┬───┬───┬───┐
 │ C │ C │ C │ C │
 └───┴───┴───┴───┘
        │
        ▼
   PostgreSQL
```

Do not create a new database connection for every request.

---

# 111. Data Security

Sensitive fields should be minimized.

Database should never contain:

```text
storage access keys
JWT signing secrets
Redis passwords
cloud credentials
raw user passwords
```

Use:

```text
environment configuration
secret management
service identities
```

---

# 112. SQL Injection Protection

All user-controlled values must use parameterized queries.

Bad:

```text
"SELECT * FROM media WHERE id = '" + id + "'"
```

Good:

```text
SELECT *
FROM media
WHERE id = $1
```

---

# 113. Schema Evolution

Schema versions should evolve independently from application versions where possible.

Example:

```text
Application v1
    │
Database schema v5

Application v2
    │
Database schema v6
```

Use backward-compatible migrations for rolling deployments.

---

# 114. Production Database Topology

Initial:

```text
              ┌──────────────┐
              │ Node.js API  │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │  PostgreSQL  │
              └──────────────┘
```

Future:

```text
                 ┌─────────────┐
                 │ Load Balancer│
                 └──────┬──────┘
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
        Node.js API            Node.js API
             │                     │
             └──────────┬──────────┘
                        ▼
                 PostgreSQL
                        │
                  ┌─────┴─────┐
                  ▼           ▼
                Primary      Replica
```

Read replicas can be introduced when required.

---

# 115. Scaling Strategy

Start:

```text
1 PostgreSQL
1 Redis
1 Node.js
1 C++ worker
1 Python worker
```

Scale to:

```text
1 PostgreSQL primary
N Node.js instances
N C++ workers
N Python workers
Redis cluster
Object storage
```

Database optimization should follow measured bottlenecks.

---

# 116. Final Schema

The initial production-oriented schema is:

```text
users
 │
 └── media
      │
      ├── media_assets
      │
      ├── artifacts
      │
      ├── jobs
      │    │
      │    ├── job_dependencies
      │    │
      │    ├── job_events
      │    │
      │    ├── processing_results
      │    │
      │    └── processing_manifests
      │
      ├── transcripts
      │    └── transcript_segments
      │
      ├── detections
      │
      ├── scenes
      │
      ├── clips
      │
      └── ai_insights

workers
 │
 └── worker_heartbeats
```

---

# 117. Architecture-to-Database Mapping

| Architecture Component     | Database Entities                   |
| -------------------------- | ----------------------------------- |
| Authentication             | users                               |
| Media API                  | media                               |
| Object storage integration | media_assets                        |
| C++ processing             | jobs, artifacts, processing_results |
| Python processing          | jobs, processing_results            |
| Queue orchestration        | jobs                                |
| Job history                | job_events                          |
| Worker monitoring          | workers, worker_heartbeats          |
| Audio transcription        | transcripts, transcript_segments    |
| Face/object detection      | detections                          |
| Scene detection            | scenes                              |
| Automatic clipping         | clips, artifacts                    |
| AI analysis                | ai_insights                         |
| Processing contract        | processing_manifests                |

---

# 118. Recommended Repository Structure

The project should eventually contain:

```text
database/
├── migrations/
│   ├── 001_extensions.sql
│   ├── 002_users.sql
│   ├── 003_media.sql
│   ├── 004_media_assets.sql
│   ├── 005_workers.sql
│   ├── 006_jobs.sql
│   ├── 007_job_dependencies.sql
│   ├── 008_job_events.sql
│   ├── 009_worker_heartbeats.sql
│   ├── 010_artifacts.sql
│   ├── 011_processing_results.sql
│   ├── 012_processing_manifests.sql
│   ├── 013_transcripts.sql
│   ├── 014_transcript_segments.sql
│   ├── 015_detections.sql
│   ├── 016_scenes.sql
│   ├── 017_clips.sql
│   ├── 018_ai_insights.sql
│   └── 019_indexes.sql
│
├── seeds/
│   ├── development.sql
│   └── demo.sql
│
├── queries/
│   ├── media.sql
│   ├── jobs.sql
│   ├── transcripts.sql
│   └── analytics.sql
│
└── README.md
```

---

# 119. Data Model Acceptance Criteria

The database design is considered complete for MVP when:

* [ ] Users can be persisted.
* [ ] Media belongs to users.
* [ ] Media assets reference object storage.
* [ ] Jobs belong to media.
* [ ] Jobs support retries.
* [ ] Jobs support parent/dependency relationships.
* [ ] Job lifecycle events are persisted.
* [ ] Workers can register.
* [ ] Worker heartbeats can be recorded.
* [ ] Processing results can be persisted.
* [ ] Transcripts can be stored.
* [ ] Transcript segments support timestamps.
* [ ] Detections support bounding boxes and timestamps.
* [ ] Scenes support time ranges.
* [ ] Generated clips can reference artifacts.
* [ ] AI insights can be stored.
* [ ] Processing manifests can be stored.
* [ ] Idempotency is supported.
* [ ] Soft deletion is supported for media.
* [ ] Major query paths are indexed.
* [ ] Database migrations are version controlled.
* [ ] Sensitive credentials are not stored in PostgreSQL.
* [ ] Large binary files are stored outside PostgreSQL.

---

# 120. Step 3 Final Architecture Decision

The database follows this principle:

```text
                 ┌────────────────────────┐
                 │       PostgreSQL       │
                 │                        │
                 │ Durable Source of Truth│
                 └───────────┬────────────┘
                             │
        ┌────────────────────┼─────────────────────┐
        │                    │                     │
        ▼                    ▼                     ▼
     Metadata             Job State             Results
        │                    │                     │
        ▼                    ▼                     ▼
      Media                Jobs               Analytics
```

While:

```text
Redis
  =
Fast coordination

Object Storage
  =
Large binary data

PostgreSQL
  =
Durable structured state
```

The most important architectural rule is:

> **PostgreSQL stores the truth about the media and processing system; Redis coordinates work; object storage stores the media itself.**

This separation allows the platform to scale C++, Python, Node.js, and storage independently without turning the database into a bottleneck.

---

# 121. Final ERD Concept

```text
                           ┌───────────────┐
                           │     USERS     │
                           ├───────────────┤
                           │ id PK         │
                           │ email         │
                           │ role          │
                           │ created_at    │
                           └───────┬───────┘
                                   │
                                   │ 1:N
                                   ▼
                           ┌───────────────┐
                           │     MEDIA     │
                           ├───────────────┤
                           │ id PK         │
                           │ user_id FK    │
                           │ title         │
                           │ media_type    │
                           │ status        │
                           │ duration_ms   │
                           └───┬───┬───┬───┘
                               │   │   │
             ┌─────────────────┘   │   └──────────────────┐
             │                     │                      │
             ▼                     ▼                      ▼
      ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
      │MEDIA_ASSETS  │      │    JOBS      │      │  TRANSCRIPTS │
      ├──────────────┤      ├──────────────┤      ├──────────────┤
      │ id           │      │ id           │      │ id           │
      │ media_id FK  │      │ media_id FK  │      │ media_id FK  │
      │ object_key   │      │ job_type     │      │ language     │
      │ checksum     │      │ status       │      │ full_text    │
      └──────────────┘      │ worker_id    │      └──────┬───────┘
                            └──────┬───────┘             │
                                   │                     ▼
                    ┌──────────────┼──────────┐  ┌──────────────────┐
                    │              │          │  │TRANSCRIPT_SEGMENTS│
                    ▼              ▼          ▼  └──────────────────┘
             ┌────────────┐ ┌────────────┐ ┌────────────┐
             │JOB_EVENTS  │ │ ARTIFACTS  │ │  RESULTS   │
             └────────────┘ └────────────┘ └────────────┘
                                   │
                   ┌───────────────┼────────────────┐
                   │               │                │
                   ▼               ▼                ▼
              ┌─────────┐    ┌───────────┐    ┌────────────┐
              │  CLIPS  │    │  SCENES   │    │DETECTIONS  │
              └─────────┘    └───────────┘    └────────────┘

                           ┌───────────────┐
                           │  AI_INSIGHTS  │
                           └───────────────┘

                           ┌───────────────┐
                           │    WORKERS    │
                           └───────┬───────┘
                                   │
                                   ▼
                           ┌───────────────┐
                           │ HEARTBEATS    │
                           └───────────────┘
```

---

# 122. Step 3 Completion

The platform now has three foundational architecture documents:

```text
docs/
├── 01-product-requirements.md
├── 02-architecture.md
└── 03-data-model.md
```

The dependency chain is:

```text
Product Requirements
        │
        ▼
System Architecture
        │
        ▼
Data Model
        │
        ▼
API Design
```

Therefore, the **next logical step is Step 4 — API Reference & Contract Design**.

Step 4 will define the actual contracts between:

```text
React
   ↕
Node.js API
   ↕
PostgreSQL
   ↕
Redis
   ↕
C++ Workers
   ↕
Python Workers
```

including authentication APIs, media upload APIs, job APIs, WebSocket events, worker APIs, request/response JSON schemas, error codes, pagination, idempotency, API versioning, and the OpenAPI structure.
