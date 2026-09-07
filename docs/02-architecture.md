# High-Performance Distributed Media Analytics Platform

## System Architecture Document

**Document:** `02-architecture.md`
**Version:** 1.0
**Status:** Draft / Foundation
**Author:** Adarsh Kumar

---

# 1. Architecture Overview

The platform follows a **polyglot distributed architecture**.

Each technology is responsible for the workload for which it is best suited:

```text
React + TypeScript
        │
        │ HTTPS / WebSocket
        ▼
Node.js + TypeScript
        │
        ├──────────────► PostgreSQL
        │
        ├──────────────► Object Storage
        │
        └──────────────► Redis
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              C++ Workers         Python Workers
                    │                   │
                    ▼                   ▼
              FFmpeg Engine         AI/ML Engine
```

The architecture separates:

* User interaction
* API orchestration
* Persistent state
* Binary storage
* Job coordination
* Native media processing
* AI/ML processing

This separation allows individual components to scale independently.

---

# 2. Architecture Goals

The architecture shall satisfy the following goals.

## 2.1 Performance

CPU-intensive media operations must not execute inside the Node.js request process.

## 2.2 Scalability

Additional C++ or Python workers should be deployable without modifying the frontend.

## 2.3 Reliability

A worker failure should not automatically result in permanent job loss.

## 2.4 Maintainability

Each major subsystem should have a clear responsibility.

## 2.5 Security

Untrusted media must be isolated from the public API.

## 2.6 Extensibility

Additional AI models and processing pipelines should be addable without redesigning the entire system.

## 2.7 Observability

A single job should be traceable across:

```text
API
 → Queue
 → Worker
 → Processing
 → Database
 → Frontend
```

---

# 3. Architecture Principles

## Principle 1 — Separate control plane and data plane

The API is primarily a **control plane**.

Workers form the **data-processing plane**.

```text
Control Plane
─────────────
React
Node.js
PostgreSQL
Redis

Data Plane
──────────
C++ Workers
Python Workers
Object Storage
```

The API should not directly perform long-running media processing.

---

## Principle 2 — Asynchronous processing

Media operations can take seconds or minutes.

Therefore:

```text
HTTP Request
     │
     ▼
Create Job
     │
     ▼
Return Job ID
     │
     ▼
Background Processing
```

The user does not keep an HTTP connection open while a video is processed.

---

## Principle 3 — Object storage for binary data

Large media files should not be stored directly inside PostgreSQL.

Instead:

```text
PostgreSQL
    │
    └── storage_key
             │
             ▼
       Object Storage
             │
             └── video.mp4
```

---

## Principle 4 — Workers are disposable

A worker should be treated as replaceable.

```text
Worker 1
   │
   X crash

Worker 2
   │
   ▼
Continue/Retry Job
```

Persistent job state must exist outside the worker process.

---

## Principle 5 — Idempotent processing

Where possible, repeating a job should not corrupt existing results.

Example:

```text
Job: thumbnail generation

First execution
    ↓
thumbnail.jpg

Retry
    ↓
same deterministic artifact
```

---

# 4. High-Level System

```text
                                  INTERNET
                                      │
                                      ▼
                         ┌────────────────────────┐
                         │ React Media Studio     │
                         │ React + TypeScript     │
                         └───────────┬────────────┘
                                     │
                         HTTPS / WebSocket
                                     │
                                     ▼
                         ┌────────────────────────┐
                         │ Node.js API Gateway     │
                         │ TypeScript              │
                         └───────────┬────────────┘
                                     │
              ┌──────────────────────┼─────────────────────┐
              │                      │                     │
              ▼                      ▼                     ▼
       ┌────────────┐        ┌─────────────┐       ┌──────────────┐
       │ PostgreSQL │        │    Redis    │       │Object Storage│
       │            │        │             │       │              │
       │ Metadata   │        │ Job Queue   │       │ Media Files  │
       │ Job State  │        │ Events      │       │ Clips        │
       └────────────┘        └──────┬──────┘       │ Thumbnails   │
                                    │              └──────────────┘
                           ┌────────┴────────┐
                           │                 │
                           ▼                 ▼
                   ┌──────────────┐   ┌──────────────┐
                   │ C++ Worker   │   │Python Worker │
                   │              │   │              │
                   │ FFmpeg       │   │ AI/ML        │
                   │ Decode       │   │ NLP          │
                   │ Encode       │   │ Vision       │
                   │ Audio        │   │ Speech       │
                   └──────────────┘   └──────────────┘
```

---

# 5. Component Architecture

The system consists of eight major components.

| Component           | Responsibility          |
| ------------------- | ----------------------- |
| React Studio        | User interface          |
| Node Gateway        | API and orchestration   |
| PostgreSQL          | Persistent metadata     |
| Redis               | Job coordination        |
| Object Storage      | Binary media            |
| C++ Engine          | Native media processing |
| Python Engine       | AI/ML processing        |
| Observability Stack | Logs, metrics, traces   |

---

# 6. Frontend Architecture

Technology:

```text
React
TypeScript
Vite
```

Recommended structure:

```text
frontend-react/
├── src/
│   ├── app/
│   ├── components/
│   ├── features/
│   │   ├── auth/
│   │   ├── dashboard/
│   │   ├── media/
│   │   ├── jobs/
│   │   ├── analytics/
│   │   └── studio/
│   ├── hooks/
│   ├── lib/
│   ├── services/
│   ├── stores/
│   ├── types/
│   └── main.tsx
├── public/
├── tests/
└── package.json
```

---

# 7. React Application Layers

```text
UI Components
      │
      ▼
Feature Modules
      │
      ▼
Application State
      │
      ▼
API Services
      │
      ▼
Node.js Gateway
```

---

# 8. React Media Studio

The primary application experience is the Media Studio.

```text
┌──────────────────────────────────────────────────────────┐
│ Header                                                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│                  Video Player                            │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                    Timeline                              │
│ ─────────────────────────────────────────────────────────│
│  Scene     Transcript       Detection                    │
├───────────────────────┬──────────────────────────────────┤
│ Transcript            │ AI Insights                      │
│                       │                                  │
│ Speaker 1: ...        │ Summary                          │
│ Speaker 2: ...        │ Topics                           │
│                       │ Sentiment                        │
└───────────────────────┴──────────────────────────────────┘
```

---

# 9. Node.js Gateway Architecture

The Node.js service acts as the application gateway.

```text
gateway-node/
├── src/
│   ├── app/
│   ├── config/
│   ├── controllers/
│   ├── middleware/
│   ├── routes/
│   ├── services/
│   ├── repositories/
│   ├── queue/
│   ├── storage/
│   ├── websocket/
│   ├── validation/
│   ├── auth/
│   ├── errors/
│   ├── logging/
│   └── server.ts
├── tests/
└── package.json
```

---

# 10. Node.js Responsibilities

Node.js shall handle:

```text
Authentication
Authorization
Media API
Job API
User API
Storage URLs
Queue submission
Job state queries
WebSocket communication
Request validation
Rate limiting
Database access
Audit events
```

Node.js should NOT:

```text
Decode large video files
Run expensive ML models
Perform long-running CPU-heavy processing
```

---

# 11. Request Lifecycle

Example: create processing job.

```text
React
 │
 │ POST /api/v1/jobs
 ▼
Node.js
 │
 ├── Authenticate
 │
 ├── Authorize
 │
 ├── Validate request
 │
 ├── Create DB job
 │
 └── Push queue message
 │
 ▼
Redis
 │
 ▼
C++ Worker
```

Node returns:

```json
{
  "id": "job_123",
  "status": "QUEUED"
}
```

---

# 12. C++ Media Engine

The C++ service is the performance-critical subsystem.

Recommended structure:

```text
engine-cpp/
├── include/
│   ├── engine/
│   ├── ffmpeg/
│   ├── media/
│   ├── processing/
│   ├── audio/
│   ├── video/
│   ├── pipeline/
│   └── common/
│
├── src/
│   ├── engine/
│   ├── ffmpeg/
│   ├── media/
│   ├── processing/
│   ├── audio/
│   ├── video/
│   ├── pipeline/
│   └── main.cpp
│
├── tests/
├── benchmarks/
├── CMakeLists.txt
└── README.md
```

---

# 13. C++ Internal Architecture

```text
                         MediaEngine
                              │
               ┌──────────────┼──────────────┐
               │              │              │
               ▼              ▼              ▼
         InputManager     JobManager     OutputManager
               │
               ▼
            Demuxer
               │
        ┌──────┴──────┐
        ▼             ▼
     Decoder       AudioDecoder
        │             │
        ▼             ▼
 VideoPipeline    AudioPipeline
        │             │
   ┌────┼────┐        ├── Resampler
   │    │    │        ├── Downmixer
   ▼    ▼    ▼        └── Encoder
Frame  Clip Thumbnail
Process
```

---

# 14. FFmpeg Integration

The C++ engine shall use FFmpeg's native libraries rather than treating the FFmpeg CLI as the primary processing interface.

Core components:

```text
libavformat
    ↓
Container / Demux / Mux

libavcodec
    ↓
Decode / Encode

libavutil
    ↓
Core media utilities

libavfilter
    ↓
Media filtering

libswscale
    ↓
Video scaling / pixel conversion

libswresample
    ↓
Audio resampling / mixing
```

FFmpeg officially provides these libraries as part of its multimedia framework.

---

# 15. C++ Media Pipeline

```text
Input File
    │
    ▼
AVFormatContext
    │
    ▼
Stream Discovery
    │
    ├──────────────┐
    ▼              ▼
Video Stream    Audio Stream
    │              │
    ▼              ▼
AVCodecContext  AVCodecContext
    │              │
    ▼              ▼
Packets         Packets
    │              │
    ▼              ▼
Frames          Samples
    │              │
    ▼              ▼
Video Pipeline  Audio Pipeline
```

FFmpeg's codec APIs use packet/frame processing concepts such as sending packets to a decoder and receiving decoded frames.

---

# 16. C++ Processing Pipeline

Initial processing:

```text
Probe
  ↓
Decode
  ↓
Extract Metadata
  ↓
Extract Frames
  ↓
Generate Thumbnail
  ↓
Generate Clip
  ↓
Extract Audio
  ↓
Normalize Audio
  ↓
Generate Processing Manifest
```

---

# 17. Python Analytics Architecture

Recommended structure:

```text
analytics-py/
├── app/
│   ├── workers/
│   ├── pipelines/
│   ├── models/
│   ├── inference/
│   ├── audio/
│   ├── vision/
│   ├── nlp/
│   ├── transcription/
│   ├── storage/
│   ├── queue/
│   ├── schemas/
│   └── config/
│
├── tests/
├── benchmarks/
├── requirements.txt
└── README.md
```

---

# 18. Python Worker Architecture

```text
Python Worker
      │
      ▼
Job Consumer
      │
      ▼
Job Validator
      │
      ▼
Media Loader
      │
      ├──────────────┐
      ▼              ▼
Audio Pipeline    Vision Pipeline
      │              │
      ▼              ▼
Speech Model      Vision Models
      │              │
      └──────┬───────┘
             ▼
        NLP Pipeline
             │
             ▼
       Result Aggregator
             │
             ▼
       Result Writer
```

---

# 19. C++ → Python Boundary

The C++ and Python systems should not initially share raw process memory.

Instead, use explicit artifacts and metadata.

```text
C++ Worker
    │
    ├── metadata.json
    ├── thumbnail.jpg
    ├── audio.wav
    ├── frames/
    └── clips/
             │
             ▼
       Object Storage
             │
             ▼
       Python Worker
```

This creates a stable boundary between native processing and AI/ML.

---

# 20. Processing Manifest

The C++ engine should produce a machine-readable manifest.

Example:

```json
{
  "job_id": "job_123",
  "media_id": "media_123",
  "source": {
    "duration_ms": 3600000,
    "width": 1920,
    "height": 1080,
    "fps": 29.97
  },
  "artifacts": {
    "audio": "media_123/audio.wav",
    "thumbnail": "media_123/thumbnail.jpg"
  }
}
```

The Python worker consumes this manifest.

---

# 21. Queue Architecture

Redis acts as the initial job coordination layer.

```text
                   Redis
                     │
       ┌─────────────┼──────────────┐
       │             │              │
       ▼             ▼              ▼
 media.jobs     ai.jobs        clip.jobs
       │             │              │
       ▼             ▼              ▼
 C++ Workers    Python Workers   C++ Workers
```

Redis Streams and consumer groups are suitable for distributed background processing patterns where workers consume jobs and processing state/recovery must be considered.

---

# 22. Queue Message

Example:

```json
{
  "job_id": "job_123",
  "media_id": "media_123",
  "type": "MEDIA_ANALYSIS",
  "priority": "NORMAL",
  "attempt": 1,
  "created_at": "2026-09-07T12:00:00Z"
}
```

Workers must validate queue messages before execution.

---

# 23. Queue Flow

```text
Node.js
   │
   │ enqueue
   ▼
Redis
   │
   │ claim
   ▼
Worker
   │
   ├── success ──► Complete
   │
   ├── retry ────► Queue Again
   │
   └── failure ──► Failed/DLQ
```

---

# 24. Worker Lifecycle

Each worker follows:

```text
START
  ↓
INITIALIZE
  ↓
CONNECT
  ↓
HEALTHY
  ↓
WAIT FOR JOB
  ↓
CLAIM JOB
  ↓
PROCESS
  ↓
REPORT PROGRESS
  ↓
COMPLETE
  ↓
ACKNOWLEDGE
  ↓
WAIT FOR JOB
```

On fatal error:

```text
PROCESS
  ↓
ERROR
  ↓
REPORT FAILURE
  ↓
RETRY OR DEAD LETTER
```

---

# 25. Job Ownership

Every running job should have:

```text
job_id
worker_id
attempt
started_at
heartbeat
status
progress
stage
```

Example:

```json
{
  "job_id": "job_123",
  "worker_id": "cpp-worker-02",
  "status": "RUNNING",
  "progress": 42,
  "stage": "FRAME_PROCESSING"
}
```

---

# 26. Worker Heartbeat

Workers should periodically update their liveness.

```text
Worker
  │
  ├── heartbeat
  ├── heartbeat
  ├── heartbeat
  └── heartbeat
```

If heartbeat stops:

```text
No heartbeat
     ↓
Worker considered unhealthy
     ↓
Job recovery mechanism
     ↓
Retry job
```

---

# 27. PostgreSQL Architecture

PostgreSQL is the system of record for application metadata.

```text
PostgreSQL
│
├── Users
├── Media
├── Media Assets
├── Jobs
├── Job Events
├── Processing Results
├── Transcripts
├── Detections
├── Scenes
├── Clips
└── AI Insights
```

---

# 28. Object Storage Architecture

Object storage contains large binary data.

```text
bucket/
│
├── users/
│   └── {user_id}/
│       └── media/
│           └── {media_id}/
│               ├── original/
│               │   └── source.mp4
│               │
│               ├── thumbnails/
│               │   └── preview.jpg
│               │
│               ├── audio/
│               │   └── normalized.wav
│               │
│               ├── clips/
│               │   ├── clip-001.mp4
│               │   └── clip-002.mp4
│               │
│               └── manifests/
│                   └── processing.json
```

---

# 29. Storage Access Model

The API should avoid unnecessarily proxying large files.

Preferred architecture:

```text
React
  │
  │ request upload URL
  ▼
Node.js
  │
  │ generate signed URL
  ▼
Object Storage
  ▲
  │
  │ direct upload
React
```

For downloads:

```text
React
  │
  ▼
Node.js
  │
  ▼
Signed Download URL
  │
  ▼
Object Storage
```

This reduces load on the API server.

---

# 30. Media Upload Flow

```text
1. React requests upload
          ↓
2. Node authenticates user
          ↓
3. Node creates media record
          ↓
4. Node generates upload URL
          ↓
5. React uploads directly
          ↓
6. Upload completion reported
          ↓
7. Node validates media
          ↓
8. Processing job created
          ↓
9. Job queued
```

---

# 31. End-to-End Processing Flow

```text
                 USER
                  │
                  ▼
              React UI
                  │
                  ▼
             Node Gateway
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
    PostgreSQL  Redis   Object Store
                  │
                  ▼
              C++ Worker
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
     Metadata   Audio     Video
        │         │          │
        └─────────┼──────────┘
                  ▼
          Processing Manifest
                  │
                  ▼
            Python Worker
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
 Transcription  Vision     NLP
       │          │          │
       └──────────┼──────────┘
                  ▼
             AI Results
                  │
          ┌───────┴────────┐
          ▼                ▼
      PostgreSQL       Object Store
          │
          ▼
      Node Gateway
          │
       WebSocket
          │
          ▼
       React UI
```

---

# 32. Job Dependency Graph

Not every task should run sequentially.

Example:

```text
                    Media
                      │
                   Probe
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Metadata    Thumbnail    Audio
                                  │
                                  ▼
                            Transcription
                                  │
          ┌───────────────────────┼──────────────┐
          ▼                       ▼              ▼
      Sentiment                Summary        Search Data
```

This allows independent work to run concurrently.

---

# 33. Recommended Job Types

```text
MEDIA_PROBE
MEDIA_DECODE
THUMBNAIL_GENERATION
AUDIO_EXTRACTION
AUDIO_NORMALIZATION
CLIP_GENERATION
TRANSCRIPTION
FACE_DETECTION
OBJECT_DETECTION
SCENE_DETECTION
SENTIMENT_ANALYSIS
SUMMARY_GENERATION
INDEXING
```

---

# 34. Pipeline Orchestration

A high-level processing job can create child jobs.

```text
MEDIA_ANALYSIS
       │
       ├── MEDIA_PROBE
       │
       ├── THUMBNAIL_GENERATION
       │
       ├── AUDIO_EXTRACTION
       │
       ├── TRANSCRIPTION
       │
       ├── FACE_DETECTION
       │
       ├── SCENE_DETECTION
       │
       └── SUMMARY_GENERATION
```

The parent job becomes complete when required child jobs finish.

---

# 35. Job State Machine

```text
                   CREATED
                      │
                      ▼
                   QUEUED
                      │
                      ▼
                   RUNNING
                 /    │    \
                /     │     \
               ▼      ▼      ▼
        COMPLETED   FAILED  CANCELLED
                     │
                     ▼
                   RETRY
                     │
                     ▼
                   QUEUED
```

---

# 36. Failure Isolation

A failure in one subsystem should not automatically crash the entire platform.

Example:

```text
Python Model Failure
       │
       X
       │
       ▼
Python Worker
retries/fails job
       │
       ▼
Node Gateway
remains healthy
```

Likewise:

```text
C++ Worker Crash
       │
       X
       │
       ▼
Redis Job State
       │
       ▼
Another C++ Worker
       │
       ▼
Retry
```

---

# 37. API and Worker Separation

The following must remain separate:

```text
Public API
     │
     X
     │
Direct execution of arbitrary native processing
```

Instead:

```text
Public API
     │
     ▼
Validated Job
     │
     ▼
Queue
     │
     ▼
Worker
```

This protects the API layer from expensive and potentially unsafe media operations.

---

# 38. Security Boundary

The architecture should establish trust zones.

```text
                 INTERNET
                     │
              UNTRUSTED ZONE
                     │
                     ▼
              API Gateway
                     │
            TRUSTED CONTROL ZONE
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
      Database     Redis     Storage
                                  │
                           Media Processing
                                  │
                          RESTRICTED ZONE
                           ┌──────┴──────┐
                           ▼             ▼
                        C++ Worker   Python Worker
```

Uploaded media should be considered untrusted input.

---

# 39. C++ Worker Security

C++ workers process potentially malicious files.

Therefore they should eventually use:

* Non-root execution
* Restricted filesystem
* Resource limits
* CPU limits
* Memory limits
* Network restrictions
* Temporary working directories
* Container isolation
* Read-only application filesystem where possible

---

# 40. Python Worker Security

Python workers should similarly use:

* Non-root user
* Dependency isolation
* Resource limits
* Restricted network access
* Temporary storage
* Model integrity verification
* Controlled filesystem permissions

---

# 41. Observability Architecture

The platform should produce:

```text
                 Applications
                      │
       ┌──────────────┼───────────────┐
       ▼              ▼               ▼
      Logs          Metrics         Traces
       │              │               │
       └──────────────┼───────────────┘
                      ▼
               Observability
                  Platform
```

---

# 42. Structured Logging

Every service should emit structured logs.

Example:

```json
{
  "timestamp": "2026-09-07T12:00:00Z",
  "level": "INFO",
  "service": "cpp-worker",
  "event": "job.completed",
  "job_id": "job_123",
  "media_id": "media_123",
  "duration_ms": 8421
}
```

---

# 43. Correlation IDs

A request should have a correlation identifier.

```text
Request
  │
  │ request_id=req_123
  ▼
Node
  │
  │ job_id=job_123
  ▼
Redis
  │
  ▼
C++ Worker
  │
  │ request_id=req_123
  ▼
Results
```

This makes debugging distributed failures much easier.

---

# 44. Metrics

Initial metrics:

```text
HTTP
────
http_requests_total
http_request_duration_seconds
http_errors_total

Queue
─────
queue_depth
queue_wait_time
jobs_enqueued_total

Workers
───────
worker_jobs_total
worker_failures_total
worker_processing_seconds
worker_cpu_usage
worker_memory_usage

Media
─────
media_processed_total
media_processing_duration
frames_processed_total

AI
──
inference_total
inference_duration
model_failures_total
```

---

# 45. Deployment Architecture — Local

The first environment should be developer-friendly.

```text
Developer Machine
│
├── React
├── Node.js
├── C++ Worker
├── Python Worker
│
└── Docker Compose
    ├── PostgreSQL
    ├── Redis
    └── Object Storage
```

---

# 46. Deployment Architecture — Containerized

```text
Docker Network
│
├── frontend
├── gateway
├── cpp-worker
├── python-worker
├── postgres
├── redis
└── object-storage
```

The exact deployment model may later be split between native development and containerized production workloads.

---

# 47. Future Cloud Architecture

The architecture should eventually support:

```text
                         CDN
                          │
                          ▼
                    React Frontend
                          │
                          ▼
                    Load Balancer
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          API-1        API-2         API-3
             │            │            │
             └────────────┼────────────┘
                          │
                    Message Queue
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
      C++ Workers     C++ Workers     Python Workers
          │               │                │
          └───────────────┼────────────────┘
                          │
                ┌─────────┴─────────┐
                ▼                   ▼
           PostgreSQL          Object Storage
```

---

# 48. Horizontal Scaling

API:

```text
             Load Balancer
            /      |      \
           ▼       ▼       ▼
        API-01   API-02   API-03
```

C++:

```text
              Queue
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
   C++-01    C++-02    C++-03
```

Python:

```text
              AI Queue
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
    PY-01      PY-02     PY-03
```

---

# 49. Independent Scaling

A major architectural benefit is independent scaling.

If video processing becomes the bottleneck:

```text
C++ Workers
2 → 10
```

without necessarily increasing:

```text
API Workers
2 → 2
```

If AI inference becomes the bottleneck:

```text
Python Workers
2 → 20
```

while the API remains unchanged.

---

# 50. Backpressure

The system must prevent workers from being overwhelmed.

```text
Incoming Jobs
      │
      ▼
    Queue
      │
      │ controlled consumption
      ▼
 Workers
```

Queue depth should be monitored.

Example:

```text
queue_depth > threshold
        │
        ▼
Scale Workers
```

---

# 51. Resource Management

Media processing can consume substantial:

* CPU
* RAM
* Disk I/O
* Network bandwidth
* GPU memory

Workers therefore need resource limits.

Example conceptual configuration:

```yaml
resources:
  cpu: "2"
  memory: "4Gi"
  temporary_storage: "10Gi"
```

Exact limits will depend on deployment environment.

---

# 52. Temporary File Strategy

Workers should use isolated temporary directories.

```text
/tmp/media-worker/
    └── job_123/
        ├── source
        ├── frames
        ├── audio
        └── output
```

After successful completion:

```text
job_123/
    ↓
cleanup
```

After failure:

```text
job_123/
    ↓
cleanup / forensic retention policy
```

The retention policy will be defined later.

---

# 53. Memory Strategy

The C++ engine should avoid loading an entire large video into memory.

Prefer:

```text
File
 ↓
Packet
 ↓
Frame
 ↓
Process
 ↓
Release
 ↓
Next Packet
```

instead of:

```text
Entire 20 GB video
        ↓
RAM
```

This is especially important for long/high-resolution media.

---

# 54. Streaming-Oriented Processing

Where practical:

```text
Input
  ↓
Demux
  ↓
Decode
  ↓
Process
  ↓
Output
```

rather than unnecessary intermediate copies.

The C++ layer should carefully manage ownership and lifetime of FFmpeg buffers.

---

# 55. CPU Parallelism

Independent operations may run concurrently.

Example:

```text
Video Decode
     │
     ├── Frame Extraction
     │
     └── Metadata

Audio Decode
     │
     └── Audio Processing
```

However, parallelism must be controlled to prevent CPU oversubscription.

---

# 56. AI Pipeline Parallelism

After media preparation:

```text
              Prepared Media
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
 Transcription    Vision       Scene
       │            │            │
       └────────────┼────────────┘
                    ▼
               Aggregation
                    │
                    ▼
                 Summary
```

This allows independent AI operations to execute concurrently.

---

# 57. Caching Strategy

Potential caches:

```text
Media Metadata
Model Metadata
Signed URLs
Frequently accessed analytics
```

Redis may eventually support caching in addition to job coordination.

However, cache data must never become the only authoritative copy of critical application state.

---

# 58. API Rate Limiting

Rate limiting should exist at the API boundary.

Example:

```text
Upload:
10 requests/minute

Job Creation:
30 requests/minute

General API:
100 requests/minute
```

Actual limits should be configurable.

---

# 59. Database Access Pattern

The Node.js service should use repository/service separation.

```text
Controller
    │
    ▼
Service
    │
    ▼
Repository
    │
    ▼
PostgreSQL
```

Example:

```text
JobController
      ↓
JobService
      ↓
JobRepository
      ↓
PostgreSQL
```

This avoids putting database logic directly inside HTTP controllers.

---

# 60. Event Architecture

Important system events:

```text
media.created
media.uploaded
media.validated
job.created
job.queued
job.started
job.progress
job.completed
job.failed
job.retried
job.cancelled
artifact.created
analytics.completed
```

These events should eventually form the basis of a consistent internal event model.

---

# 61. WebSocket Architecture

```text
                    Node.js
                       │
                 WebSocket Manager
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Client A      Client B     Client C
```

A client subscribes to:

```text
/ws/jobs/{jobId}
```

Events:

```text
job.started
job.progress
job.completed
job.failed
```

---

# 62. Realtime Event Flow

```text
C++ Worker
    │
    │ progress
    ▼
Redis / Event Layer
    │
    ▼
Node.js
    │
    ▼
WebSocket
    │
    ▼
React
```

The worker should not need to maintain a direct connection to the browser.

---

# 63. API Versioning

All public APIs should use explicit versions.

```text
/api/v1/
```

Future:

```text
/api/v2/
```

Breaking API changes should require a version change rather than silently changing existing behavior.

---

# 64. Configuration Architecture

Configuration should be externalized.

Example:

```text
NODE_ENV
DATABASE_URL
REDIS_URL
OBJECT_STORAGE_ENDPOINT
OBJECT_STORAGE_BUCKET
OBJECT_STORAGE_ACCESS_KEY
OBJECT_STORAGE_SECRET_KEY
JWT_SECRET
LOG_LEVEL
WORKER_CONCURRENCY
MAX_UPLOAD_SIZE
```

Secrets must not be committed to Git.

---

# 65. Environment Separation

```text
development
    │
    ├── local database
    ├── local Redis
    └── local object storage

testing
    │
    ├── isolated database
    ├── isolated queue
    └── test storage

production
    │
    ├── managed database
    ├── managed queue
    └── cloud object storage
```

---

# 66. Development Architecture

Development should allow individual services to be run independently.

Example:

```text
Terminal 1
──────────
npm run dev

Terminal 2
──────────
./engine-cpp/build/media-worker

Terminal 3
──────────
python -m analytics.worker

Terminal 4
──────────
docker compose up postgres redis object-storage
```

This makes debugging easier.

---

# 67. Testing Architecture

Each service has its own test suite.

```text
React
 ├── Unit
 ├── Component
 └── E2E

Node
 ├── Unit
 ├── API
 └── Integration

C++
 ├── Unit
 ├── Integration
 └── Benchmark

Python
 ├── Unit
 ├── Model
 └── Pipeline

System
 └── End-to-End
```

---

# 68. Performance Testing

The platform should eventually measure:

```text
1 minute 720p video
1 minute 1080p video
1 minute 4K video
10 minute video
1 hour video
```

Metrics:

```text
Decode FPS
Processing FPS
CPU %
RAM
Disk I/O
Queue latency
Total processing time
```

---

# 69. Benchmark Architecture

The C++ engine should have dedicated benchmarks.

```text
benchmarks/
├── decode_benchmark
├── frame_processing_benchmark
├── audio_benchmark
├── thumbnail_benchmark
├── clip_benchmark
└── pipeline_benchmark
```

The benchmark suite should measure changes across versions.

---

# 70. Architecture Trade-Offs

## Polyglot Architecture

### Advantages

* C++ performance
* Python AI ecosystem
* Node.js API productivity
* React frontend ecosystem
* Clear separation

### Disadvantages

* Multiple languages
* More deployment complexity
* Multiple build systems
* More operational overhead

Decision:

**Accept the complexity because demonstrating cross-language systems architecture is a primary project goal.**

---

# 71. Redis vs Direct Processing

### Direct processing

```text
API → Worker
```

Problems:

* Tight coupling
* No durable workload boundary
* Poor scaling
* Difficult retries

### Queue architecture

```text
API → Redis → Worker
```

Advantages:

* Decoupling
* Retry
* Worker scaling
* Backpressure
* Job visibility

Decision:

**Use Redis as the initial job coordination layer.**

---

# 72. PostgreSQL vs File-Based Metadata

### File-based metadata

Simple but difficult to query.

### PostgreSQL

Provides:

* Transactions
* Indexing
* Relationships
* Querying
* Constraints
* Concurrent access

Decision:

**Use PostgreSQL as the application metadata store.**

---

# 73. Object Storage vs Database BLOBs

### Database BLOBs

Problems:

* Database growth
* Backup complexity
* Heavy I/O
* Poor separation of concerns

### Object storage

Advantages:

* Designed for large binary objects
* Independent scaling
* Separate lifecycle management
* Direct client uploads/downloads

Decision:

**Use object storage for media and generated artifacts.**

---

# 74. Native FFmpeg vs CLI

### FFmpeg CLI

Advantages:

* Easy initial experimentation
* Fast prototyping

Disadvantages:

* Process spawning overhead
* Limited direct buffer control
* Harder integration with custom processing
* Less control over memory/data flow

### Native libraries

Advantages:

* Direct API access
* Better integration with C++
* Custom pipelines
* Better control over frames and packets
* Easier integration with custom processing

Decision:

**Use FFmpeg native libraries for the C++ engine.**

---

# 75. Initial Architecture Decision

The final initial architecture is:

```text
                React + TypeScript
                        │
                        ▼
               Node.js + TypeScript
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
     PostgreSQL       Redis      Object Storage
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
        C++ Media Workers    Python AI Workers
              │                   │
              ▼                   ▼
          FFmpeg             AI/ML Models
              │                   │
              └─────────┬─────────┘
                        ▼
                   Result Store
                        │
                        ▼
                   React Studio
```

---

# 76. Architectural Boundaries

The following boundaries must remain explicit:

```text
Frontend
   │
   │ HTTP/WebSocket
   ▼
API
   │
   │ Queue
   ▼
Workers
   │
   │ Object Storage
   ▼
Artifacts
```

The frontend must not communicate directly with internal worker processes.

Workers must not expose public APIs directly to the Internet.

The database should not be directly accessible from the frontend.

---

# 77. Dependency Direction

The preferred dependency direction is:

```text
React
  ↓
API Contract
  ↓
Node Services
  ↓
Infrastructure

Workers
  ↓
Infrastructure

Infrastructure
  ↓
External Systems
```

Internal components should avoid circular dependencies.

---

# 78. Architectural Quality Attributes

The system should be evaluated against:

| Attribute            | Target      |
| -------------------- | ----------- |
| Performance          | High        |
| Scalability          | High        |
| Reliability          | High        |
| Security             | High        |
| Maintainability      | High        |
| Observability        | High        |
| Developer Experience | High        |
| Portability          | Medium/High |
| Cost Efficiency      | Medium      |

---

# 79. Architecture Evolution

The project should evolve in stages.

```text
Stage 1
Monolithic Local Development
        ↓
Stage 2
Separated Workers
        ↓
Stage 3
Dockerized Services
        ↓
Stage 4
Horizontally Scaled Workers
        ↓
Stage 5
Cloud Deployment
        ↓
Stage 6
GPU / Kubernetes
```

We should **not start at Stage 6**.

---

# 80. Architecture Implementation Order

The architecture should be implemented in this order:

```text
1. Repository
       ↓
2. Shared contracts
       ↓
3. PostgreSQL
       ↓
4. Object storage
       ↓
5. Redis
       ↓
6. C++ media engine
       ↓
7. Python worker
       ↓
8. Node.js gateway
       ↓
9. React frontend
       ↓
10. WebSocket
       ↓
11. Integration
       ↓
12. Observability
       ↓
13. Security hardening
       ↓
14. Performance optimization
       ↓
15. Docker / cloud
```

---

# 81. Architecture Acceptance Criteria

This architecture is accepted when:

* [ ] Frontend is independent of workers.
* [ ] Node.js does not perform heavy media processing.
* [ ] C++ owns native media processing.
* [ ] Python owns AI/ML processing.
* [ ] PostgreSQL owns application metadata.
* [ ] Object storage owns binary assets.
* [ ] Redis owns asynchronous job coordination.
* [ ] Jobs have persistent states.
* [ ] Workers can be scaled independently.
* [ ] Failed jobs can be retried.
* [ ] Worker crashes can be detected.
* [ ] Real-time job updates are supported.
* [ ] Services have clear boundaries.
* [ ] Security boundaries are defined.
* [ ] Local development is possible.
* [ ] Future cloud deployment does not require a fundamental redesign.

---

# 82. Architecture Summary

The platform uses a distributed architecture built around one central idea:

```text
                  CONTROL
                    │
              Node.js API
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       Database             Queue
          │                   │
          │            ┌──────┴──────┐
          │            ▼             ▼
          │         C++ Workers   Python Workers
          │            │             │
          │            ▼             ▼
          │         FFmpeg          AI/ML
          │            │             │
          └────────────┴─────────────┘
                       │
                       ▼
                 Object Storage
                       │
                       ▼
                 React Studio
```

The architecture deliberately separates **control, storage, orchestration, media processing, and AI processing**. This provides the foundation for a system that can begin as a local development project and evolve toward a distributed production platform.
