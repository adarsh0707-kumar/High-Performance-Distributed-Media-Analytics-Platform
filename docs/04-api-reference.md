# High-Performance Distributed Media Analytics Platform

# API Reference

**Document:** `04-api-reference.md`
**Version:** 1.0
**Status:** Implementation Ready
**API Style:** REST + WebSocket
**Base Path:** `/api/v1`
**Primary Backend:** Node.js + TypeScript
**Database:** PostgreSQL
**Queue:** Redis
**Author:** Adarsh Kumar
**Last Updated:** 2026-09-07

---

# 1. Purpose

This document defines the public and internal API contracts for the High-Performance Distributed Media Analytics Platform.

The API layer is responsible for:

* authentication
* user management
* media uploads
* media metadata
* processing jobs
* job cancellation
* job status
* processing results
* transcripts
* detections
* scenes
* clips
* AI insights
* worker communication
* real-time progress updates
* health monitoring

The API gateway is implemented using:

```text
Node.js
+
TypeScript
+
HTTP REST
+
WebSocket
```

---

# 2. API Architecture

```text
                    React + TypeScript
                           │
                           │ HTTPS
                           ▼
                ┌─────────────────────┐
                │   Node.js Gateway   │
                │      /api/v1        │
                └──────────┬──────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         PostgreSQL      Redis      Object Storage
              │            │
              │            ▼
              │      ┌─────────────┐
              │      │ C++ Workers │
              │      └─────────────┘
              │
              │      ┌──────────────┐
              └──────│Python Workers│
                     └──────────────┘
```

---

# 3. Base URL

Development:

```text
http://localhost:3000/api/v1
```

Production:

```text
https://<domain>/api/v1
```

The production hostname is environment-specific.

---

# 4. API Versioning

All public API endpoints are versioned.

Current version:

```text
/api/v1
```

Example:

```text
GET /api/v1/media
```

Future incompatible changes should use:

```text
/api/v2
```

Do not silently change the meaning of an existing endpoint.

---

# 5. HTTP Methods

The API follows standard HTTP semantics.

| Method   | Purpose                          |
| -------- | -------------------------------- |
| `GET`    | Read resource                    |
| `POST`   | Create resource / execute action |
| `PUT`    | Replace resource                 |
| `PATCH`  | Partially update                 |
| `DELETE` | Delete/cancel resource           |

---

# 6. Authentication

Authentication uses access tokens.

Recommended model:

```text
Access Token
    │
    ▼
Authorization: Bearer <token>
```

Example:

```http
Authorization: Bearer eyJ...
```

Access tokens should be short-lived.

Long-lived session/refresh information is maintained server-side.

---

# 7. Authentication Endpoints

```text
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
GET  /api/v1/auth/me
```

---

# 8. Register

## Endpoint

```http
POST /api/v1/auth/register
```

## Request

```json
{
  "email": "user@example.com",
  "password": "StrongPassword123!",
  "displayName": "Adarsh Kumar"
}
```

## Validation

```text
email:
  required
  valid email format

password:
  required
  minimum 8 characters

displayName:
  required
  1–150 characters
```

## Response

```http
201 Created
```

```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "displayName": "Adarsh Kumar",
    "role": "user"
  }
}
```

Passwords are never returned.

---

# 9. Login

## Endpoint

```http
POST /api/v1/auth/login
```

## Request

```json
{
  "email": "user@example.com",
  "password": "StrongPassword123!"
}
```

## Success

```http
200 OK
```

```json
{
  "accessToken": "jwt-token",
  "expiresIn": 900,
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "displayName": "Adarsh Kumar",
    "role": "user"
  }
}
```

---

# 10. Refresh

```http
POST /api/v1/auth/refresh
```

The refresh token is preferably transmitted using a secure, HTTP-only cookie.

Response:

```json
{
  "accessToken": "new-access-token",
  "expiresIn": 900
}
```

---

# 11. Logout

```http
POST /api/v1/auth/logout
```

Response:

```http
204 No Content
```

The corresponding server-side session is revoked.

---

# 12. Current User

```http
GET /api/v1/auth/me
```

Response:

```json
{
  "id": "uuid",
  "email": "user@example.com",
  "displayName": "Adarsh Kumar",
  "role": "user",
  "createdAt": "2026-09-07T18:00:00Z"
}
```

---

# 13. Media API

Main media endpoints:

```text
POST   /api/v1/media
GET    /api/v1/media
GET    /api/v1/media/:mediaId
PATCH  /api/v1/media/:mediaId
DELETE /api/v1/media/:mediaId
```

---

# 14. Create Media Upload

The recommended upload architecture is:

```text
React
  │
  │ create upload
  ▼
Node.js
  │
  │ create media record
  ▼
PostgreSQL
  │
  │ generate upload target
  ▼
React
  │
  │ upload directly
  ▼
Object Storage
```

Endpoint:

```http
POST /api/v1/media
```

Request:

```json
{
  "title": "Product Demo",
  "filename": "demo.mp4",
  "contentType": "video/mp4",
  "sizeBytes": 104857600
}
```

Response:

```http
201 Created
```

```json
{
  "media": {
    "id": "media-uuid",
    "title": "Product Demo",
    "originalFilename": "demo.mp4",
    "mediaType": "video",
    "status": "uploading"
  },
  "upload": {
    "method": "PUT",
    "url": "signed-upload-url",
    "expiresAt": "2026-09-07T18:20:00Z"
  }
}
```

The actual object-storage URL is generated by the backend.

---

# 15. Complete Upload Flow

```text
┌──────────────┐
│ React Client │
└──────┬───────┘
       │
       │ POST /media
       ▼
┌──────────────┐
│ Node Gateway │
└──────┬───────┘
       │
       ├── INSERT media
       │
       └── Generate upload target
              │
              ▼
        React receives target
              │
              ▼
        Object Storage
              │
              ▼
        Upload completed
              │
              ▼
 POST /media/:id/complete
              │
              ▼
        Create processing job
```

---

# 16. Complete Upload

```http
POST /api/v1/media/:mediaId/complete
```

Request:

```json
{
  "sizeBytes": 104857600,
  "checksumSha256": "sha256-value"
}
```

The backend verifies the object exists.

Response:

```json
{
  "media": {
    "id": "uuid",
    "status": "ready"
  },
  "job": {
    "id": "uuid",
    "type": "MEDIA_PROBE",
    "status": "queued"
  }
}
```

---

# 17. List Media

```http
GET /api/v1/media
```

Query parameters:

```text
page
limit
status
mediaType
search
sort
order
```

Example:

```http
GET /api/v1/media?page=1&limit=20&status=processed
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Product Demo",
      "mediaType": "video",
      "status": "processed",
      "durationMs": 125000,
      "createdAt": "2026-09-07T18:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

---

# 18. Get Media

```http
GET /api/v1/media/:mediaId
```

Response:

```json
{
  "id": "uuid",
  "title": "Product Demo",
  "originalFilename": "demo.mp4",
  "mediaType": "video",
  "status": "processed",
  "mimeType": "video/mp4",
  "sizeBytes": 104857600,
  "durationMs": 125000,
  "width": 1920,
  "height": 1080,
  "frameRate": 30,
  "codec": "h264",
  "createdAt": "2026-09-07T18:00:00Z"
}
```

---

# 19. Update Media

```http
PATCH /api/v1/media/:mediaId
```

Request:

```json
{
  "title": "Updated Product Demo"
}
```

Response:

```json
{
  "id": "uuid",
  "title": "Updated Product Demo"
}
```

Only mutable metadata should be updated.

---

# 20. Delete Media

```http
DELETE /api/v1/media/:mediaId
```

The initial implementation should use soft deletion.

Response:

```http
204 No Content
```

The backend marks:

```text
deleted_at
status = deleted
```

Object-storage cleanup can happen asynchronously.

---

# 21. Media Assets API

```text
GET /api/v1/media/:mediaId/assets
GET /api/v1/media/:mediaId/assets/:assetId
GET /api/v1/media/:mediaId/assets/:assetId/url
```

---

# 22. List Assets

```http
GET /api/v1/media/:mediaId/assets
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "assetType": "source",
      "contentType": "video/mp4",
      "sizeBytes": 104857600
    },
    {
      "id": "uuid",
      "assetType": "thumbnail",
      "contentType": "image/jpeg",
      "sizeBytes": 45231
    }
  ]
}
```

---

# 23. Asset Download URL

```http
GET /api/v1/media/:mediaId/assets/:assetId/url
```

Response:

```json
{
  "url": "signed-download-url",
  "expiresAt": "2026-09-07T18:30:00Z"
}
```

The API should not expose permanent object-storage credentials.

---

# 24. Processing Jobs API

Main endpoints:

```text
POST /api/v1/media/:mediaId/jobs
GET  /api/v1/jobs
GET  /api/v1/jobs/:jobId
POST /api/v1/jobs/:jobId/cancel
POST /api/v1/jobs/:jobId/retry
GET  /api/v1/jobs/:jobId/events
```

---

# 25. Create Job

```http
POST /api/v1/media/:mediaId/jobs
```

Request:

```json
{
  "type": "TRANSCRIBE",
  "priority": 100,
  "parameters": {
    "language": "en",
    "model": "whisper"
  }
}
```

Response:

```http
202 Accepted
```

```json
{
  "id": "job-uuid",
  "mediaId": "media-uuid",
  "type": "TRANSCRIBE",
  "status": "queued",
  "priority": 100,
  "createdAt": "2026-09-07T18:00:00Z"
}
```

---

# 26. Supported Job Types

Initial job types:

| Type                 | Worker       |
| -------------------- | ------------ |
| `MEDIA_PROBE`        | C++          |
| `VIDEO_TRANSCODE`    | C++          |
| `AUDIO_EXTRACT`      | C++          |
| `THUMBNAIL_GENERATE` | C++          |
| `WAVEFORM_GENERATE`  | C++          |
| `SCENE_DETECT`       | C++ / Python |
| `FACE_DETECT`        | Python       |
| `OBJECT_DETECT`      | Python       |
| `TRANSCRIBE`         | Python       |
| `SENTIMENT_ANALYSIS` | Python       |
| `SUMMARY`            | Python       |
| `CLIP_GENERATE`      | C++          |
| `AI_ANALYSIS`        | Python       |

---

# 27. Job Status

Possible statuses:

```text
queued
leased
running
succeeded
failed
cancelled
dead_letter
```

Lifecycle:

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
          ┌──────┴──────┐
          ▼             ▼
        retry       dead_letter
```

---

# 28. Get Job

```http
GET /api/v1/jobs/:jobId
```

Response:

```json
{
  "id": "uuid",
  "mediaId": "uuid",
  "type": "TRANSCRIBE",
  "status": "running",
  "priority": 100,
  "attemptCount": 1,
  "maxAttempts": 3,
  "startedAt": "2026-09-07T18:05:00Z",
  "worker": {
    "id": "uuid",
    "type": "python",
    "version": "1.0.0"
  }
}
```

---

# 29. List Jobs

```http
GET /api/v1/jobs
```

Query parameters:

```text
page
limit
status
type
mediaId
sort
order
```

Example:

```http
GET /api/v1/jobs?status=running&type=TRANSCRIBE
```

---

# 30. Cancel Job

```http
POST /api/v1/jobs/:jobId/cancel
```

Response:

```json
{
  "id": "uuid",
  "status": "cancelled"
}
```

A running worker should receive a cancellation signal.

The worker must stop safely.

---

# 31. Retry Job

```http
POST /api/v1/jobs/:jobId/retry
```

Retry is permitted when the job is:

```text
failed
dead_letter
```

Response:

```json
{
  "id": "uuid",
  "status": "queued",
  "attemptCount": 0
}
```

Depending on implementation, retry may create a new job instead of mutating the old one.

For auditability, creating a new processing run is preferred for major reprocessing operations.

---

# 32. Job Events

```http
GET /api/v1/jobs/:jobId/events
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "eventType": "queued",
      "createdAt": "2026-09-07T18:00:00Z"
    },
    {
      "id": "uuid",
      "eventType": "started",
      "createdAt": "2026-09-07T18:01:00Z"
    },
    {
      "id": "uuid",
      "eventType": "progress",
      "progressPercent": 42.5,
      "message": "Extracting audio",
      "createdAt": "2026-09-07T18:01:30Z"
    }
  ]
}
```

---

# 33. Processing Results API

```text
GET /api/v1/jobs/:jobId/result
GET /api/v1/media/:mediaId/results
```

---

# 34. Job Result

```http
GET /api/v1/jobs/:jobId/result
```

Response:

```json
{
  "jobId": "uuid",
  "resultType": "transcription",
  "summary": {
    "language": "en",
    "durationMs": 125000,
    "segmentCount": 84
  },
  "metrics": {
    "processingTimeMs": 38200,
    "cpuTimeMs": 36100
  }
}
```

---

# 35. Media Results

```http
GET /api/v1/media/:mediaId/results
```

Response:

```json
{
  "data": [
    {
      "jobId": "uuid",
      "resultType": "probe",
      "createdAt": "2026-09-07T18:00:10Z"
    },
    {
      "jobId": "uuid",
      "resultType": "transcription",
      "createdAt": "2026-09-07T18:02:10Z"
    }
  ]
}
```

---

# 36. Transcript API

```text
GET /api/v1/media/:mediaId/transcripts
GET /api/v1/transcripts/:transcriptId
GET /api/v1/transcripts/:transcriptId/segments
```

---

# 37. Get Transcripts

```http
GET /api/v1/media/:mediaId/transcripts
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "language": "en",
      "modelName": "whisper",
      "confidence": 0.94,
      "createdAt": "2026-09-07T18:10:00Z"
    }
  ]
}
```

---

# 38. Get Transcript

```http
GET /api/v1/transcripts/:transcriptId
```

Response:

```json
{
  "id": "uuid",
  "mediaId": "uuid",
  "language": "en",
  "modelName": "whisper",
  "fullText": "Welcome to the media analytics platform...",
  "confidence": 0.94
}
```

---

# 39. Transcript Segments

```http
GET /api/v1/transcripts/:transcriptId/segments
```

Query:

```text
startMs
endMs
limit
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "segmentIndex": 0,
      "startMs": 0,
      "endMs": 4200,
      "text": "Welcome to the platform.",
      "speakerLabel": "speaker_0",
      "confidence": 0.97
    }
  ]
}
```

---

# 40. Detection API

```text
GET /api/v1/media/:mediaId/detections
GET /api/v1/media/:mediaId/detections/:detectionId
```

Query parameters:

```text
type
label
startMs
endMs
trackId
confidenceMin
confidenceMax
```

Example:

```http
GET /api/v1/media/:mediaId/detections?type=face&startMs=10000&endMs=30000
```

---

# 41. Detection Response

```json
{
  "data": [
    {
      "id": "uuid",
      "type": "face",
      "label": "person",
      "confidence": 0.981,
      "startMs": 12000,
      "endMs": 12500,
      "frameNumber": 360,
      "boundingBox": {
        "x": 0.21,
        "y": 0.14,
        "width": 0.18,
        "height": 0.32
      },
      "trackId": "track-12"
    }
  ]
}
```

---

# 42. Scene API

```text
GET /api/v1/media/:mediaId/scenes
GET /api/v1/media/:mediaId/scenes/:sceneId
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "sceneIndex": 0,
      "startMs": 0,
      "endMs": 12000,
      "score": 0.94,
      "label": "intro"
    }
  ]
}
```

---

# 43. Clip API

Main endpoints:

```text
POST   /api/v1/media/:mediaId/clips
GET    /api/v1/media/:mediaId/clips
GET    /api/v1/clips/:clipId
PATCH  /api/v1/clips/:clipId
DELETE /api/v1/clips/:clipId
POST   /api/v1/clips/:clipId/render
```

---

# 44. Create Clip

```http
POST /api/v1/media/:mediaId/clips
```

Request:

```json
{
  "name": "Important Segment",
  "startMs": 120000,
  "endMs": 165000,
  "format": "mp4"
}
```

Response:

```http
201 Created
```

```json
{
  "id": "clip-uuid",
  "mediaId": "media-uuid",
  "name": "Important Segment",
  "startMs": 120000,
  "endMs": 165000,
  "status": "requested"
}
```

---

# 45. Render Clip

```http
POST /api/v1/clips/:clipId/render
```

Response:

```http
202 Accepted
```

```json
{
  "clipId": "uuid",
  "job": {
    "id": "uuid",
    "type": "CLIP_GENERATE",
    "status": "queued"
  }
}
```

---

# 46. AI Insights API

```text
GET /api/v1/media/:mediaId/insights
GET /api/v1/media/:mediaId/insights/:insightId
```

Query:

```text
type
model
```

---

# 47. AI Insight Response

```json
{
  "data": [
    {
      "id": "uuid",
      "type": "summary",
      "modelName": "summarizer-v1",
      "confidence": 0.91,
      "content": {
        "summary": "The video demonstrates..."
      },
      "createdAt": "2026-09-07T18:20:00Z"
    }
  ]
}
```

---

# 48. Search API

The media studio requires search.

Initial endpoint:

```http
GET /api/v1/search
```

Query:

```text
q
type
mediaId
page
limit
```

Example:

```http
GET /api/v1/search?q=machine+learning
```

The initial implementation can search:

```text
media.title
media.original_filename
transcripts.full_text
transcript_segments.text
AI insight content
```

Full-text indexing can be introduced as the dataset grows.

---

# 49. Dashboard API

The React dashboard can use:

```http
GET /api/v1/dashboard
```

Response:

```json
{
  "media": {
    "total": 120,
    "processing": 4,
    "processed": 108,
    "failed": 8
  },
  "jobs": {
    "queued": 12,
    "running": 4,
    "failed": 3
  },
  "storage": {
    "bytesUsed": 107374182400
  },
  "workers": {
    "online": 5,
    "offline": 1
  }
}
```

---

# 50. Health API

Public health endpoint:

```http
GET /health
```

Response:

```json
{
  "status": "ok"
}
```

Detailed readiness endpoint:

```http
GET /health/ready
```

Response:

```json
{
  "status": "ready",
  "dependencies": {
    "postgres": "ok",
    "redis": "ok",
    "objectStorage": "ok"
  }
}
```

---

# 51. Liveness

```http
GET /health/live
```

This should only verify that the Node.js process is alive.

It should not require PostgreSQL or Redis.

Response:

```json
{
  "status": "alive"
}
```

---

# 52. Worker API

Worker endpoints are internal APIs.

```text
POST /internal/v1/workers/register
POST /internal/v1/workers/:workerId/heartbeat
POST /internal/v1/workers/:workerId/jobs/:jobId/start
POST /internal/v1/workers/:workerId/jobs/:jobId/progress
POST /internal/v1/workers/:workerId/jobs/:jobId/complete
POST /internal/v1/workers/:workerId/jobs/:jobId/fail
```

These endpoints must not be publicly accessible.

---

# 53. Worker Registration

```http
POST /internal/v1/workers/register
```

Request:

```json
{
  "workerKey": "cpp-worker-01",
  "workerType": "cpp",
  "hostname": "worker-node-01",
  "version": "1.0.0",
  "capabilities": {
    "ffmpeg": true,
    "codecs": [
      "h264",
      "h265",
      "aac"
    ]
  }
}
```

Response:

```json
{
  "workerId": "uuid",
  "status": "online"
}
```

---

# 54. Worker Heartbeat

```http
POST /internal/v1/workers/:workerId/heartbeat
```

Response:

```json
{
  "workerId": "uuid",
  "status": "online",
  "serverTime": "2026-09-07T18:20:00Z"
}
```

---

# 55. Worker Job Start

```http
POST /internal/v1/workers/:workerId/jobs/:jobId/start
```

Request:

```json
{
  "attemptNumber": 1
}
```

Response:

```json
{
  "jobId": "uuid",
  "status": "running"
}
```

---

# 56. Worker Progress

```http
POST /internal/v1/workers/:workerId/jobs/:jobId/progress
```

Request:

```json
{
  "progressPercent": 47.5,
  "message": "Decoding video frames",
  "metrics": {
    "framesProcessed": 14250,
    "fps": 238.4
  }
}
```

Response:

```json
{
  "accepted": true
}
```

The gateway persists an event in `job_events`.

---

# 57. Worker Completion

```http
POST /internal/v1/workers/:workerId/jobs/:jobId/complete
```

Request:

```json
{
  "metrics": {
    "processingTimeMs": 18200,
    "framesProcessed": 45000
  },
  "outputs": [
    {
      "assetType": "thumbnail",
      "bucket": "media",
      "objectKey": "media/uuid/thumbnail.jpg"
    }
  ]
}
```

Response:

```json
{
  "jobId": "uuid",
  "status": "succeeded"
}
```

---

# 58. Worker Failure

```http
POST /internal/v1/workers/:workerId/jobs/:jobId/fail
```

Request:

```json
{
  "errorCode": "FFMPEG_DECODE_ERROR",
  "message": "Unable to decode input stream",
  "retryable": true,
  "details": {
    "codec": "h265"
  }
}
```

Response:

```json
{
  "jobId": "uuid",
  "status": "failed",
  "retryScheduled": true
}
```

---

# 59. WebSocket API

The frontend requires real-time processing updates.

Connection:

```text
/ws
```

Authentication should occur during connection establishment.

---

# 60. WebSocket Subscription

Client:

```json
{
  "type": "subscribe",
  "channel": "job",
  "jobId": "uuid"
}
```

Server:

```json
{
  "type": "subscribed",
  "channel": "job",
  "jobId": "uuid"
}
```

---

# 61. Job Progress Event

Server:

```json
{
  "type": "job.progress",
  "jobId": "uuid",
  "status": "running",
  "progressPercent": 65.5,
  "message": "Extracting audio",
  "timestamp": "2026-09-07T18:30:00Z"
}
```

---

# 62. Job Completion Event

```json
{
  "type": "job.completed",
  "jobId": "uuid",
  "status": "succeeded",
  "timestamp": "2026-09-07T18:35:00Z"
}
```

---

# 63. Job Failure Event

```json
{
  "type": "job.failed",
  "jobId": "uuid",
  "status": "failed",
  "error": {
    "code": "TRANSCRIPTION_FAILED",
    "message": "Unable to process audio"
  },
  "timestamp": "2026-09-07T18:35:00Z"
}
```

---

# 64. WebSocket Event Types

```text
job.queued
job.leased
job.started
job.progress
job.completed
job.failed
job.cancelled
job.retrying
worker.online
worker.offline
media.updated
clip.ready
insight.created
```

---

# 65. Standard Response Envelope

Successful responses should use a consistent structure where appropriate.

Single resource:

```json
{
  "data": {
    "id": "uuid"
  }
}
```

Collection:

```json
{
  "data": [],
  "pagination": {}
}
```

However, simple action endpoints may return a direct object when an envelope provides no practical value.

---

# 66. Error Response

All API errors should follow:

```json
{
  "error": {
    "code": "MEDIA_NOT_FOUND",
    "message": "Media resource was not found.",
    "requestId": "uuid",
    "details": {}
  }
}
```

---

# 67. HTTP Status Codes

| Status | Meaning                                  |
| ------ | ---------------------------------------- |
| `200`  | Successful request                       |
| `201`  | Resource created                         |
| `202`  | Async operation accepted                 |
| `204`  | Successful request with no response body |
| `400`  | Invalid request                          |
| `401`  | Authentication required                  |
| `403`  | Permission denied                        |
| `404`  | Resource not found                       |
| `409`  | Resource conflict                        |
| `413`  | Payload too large                        |
| `415`  | Unsupported media type                   |
| `422`  | Validation failure                       |
| `429`  | Rate limited                             |
| `500`  | Internal server error                    |
| `502`  | Upstream service failure                 |
| `503`  | Service unavailable                      |

---

# 68. Error Codes

Authentication:

```text
AUTH_INVALID_CREDENTIALS
AUTH_TOKEN_EXPIRED
AUTH_TOKEN_INVALID
AUTH_SESSION_REVOKED
AUTH_EMAIL_EXISTS
```

Media:

```text
MEDIA_NOT_FOUND
MEDIA_INVALID_TYPE
MEDIA_TOO_LARGE
MEDIA_UPLOAD_EXPIRED
MEDIA_UPLOAD_INCOMPLETE
MEDIA_DELETED
```

Jobs:

```text
JOB_NOT_FOUND
JOB_INVALID_TYPE
JOB_ALREADY_RUNNING
JOB_CANCELLED
JOB_NOT_RETRYABLE
JOB_MAX_ATTEMPTS
JOB_DEPENDENCY_FAILED
```

Workers:

```text
WORKER_NOT_FOUND
WORKER_UNAUTHORIZED
WORKER_UNAVAILABLE
WORKER_HEARTBEAT_EXPIRED
```

Processing:

```text
FFMPEG_DECODE_ERROR
FFMPEG_ENCODE_ERROR
AUDIO_EXTRACTION_FAILED
TRANSCRIPTION_FAILED
MODEL_EXECUTION_FAILED
ARTIFACT_WRITE_FAILED
```

---

# 69. Request IDs

Every request receives a unique ID.

Example:

```http
X-Request-ID: 4c1f0e8a-...
```

If the client provides a valid request ID, the gateway can propagate it.

The ID must appear in:

* API logs
* database audit records where appropriate
* job correlation
* worker logs
* error responses

---

# 70. Correlation IDs

A processing workflow should have a correlation ID.

Example:

```text
Upload
 │
 └── correlationId = ABC
       │
       ├── MEDIA_PROBE
       ├── AUDIO_EXTRACT
       ├── TRANSCRIBE
       ├── SUMMARY
       └── CLIP_GENERATE
```

This allows the complete workflow to be traced.

---

# 71. Idempotency

Mutation endpoints should support:

```http
Idempotency-Key: <unique-key>
```

Especially:

```text
POST /media
POST /media/:id/jobs
POST /clips/:id/render
```

Example:

```http
Idempotency-Key: transcribe-media-123-v1
```

The backend should prevent accidental duplicate processing.

---

# 72. Pagination

Initial REST APIs use:

```text
page
limit
```

Example:

```http
GET /api/v1/jobs?page=2&limit=50
```

Default:

```text
page = 1
limit = 20
```

Maximum:

```text
limit = 100
```

For very large datasets, cursor pagination can be introduced later.

---

# 73. Filtering

Filtering uses query parameters.

Example:

```http
GET /api/v1/jobs?status=failed&type=TRANSCRIBE
```

Do not accept arbitrary SQL expressions from clients.

---

# 74. Sorting

Example:

```http
GET /api/v1/media?sort=createdAt&order=desc
```

Only whitelisted fields may be used.

Example allowed fields:

```text
createdAt
updatedAt
title
durationMs
sizeBytes
```

---

# 75. Rate Limiting

Public API endpoints should be rate-limited.

Recommended categories:

```text
Authentication
    strict

Media upload creation
    moderate

Read APIs
    higher

Job creation
    moderate

WebSocket connections
    connection limit
```

Redis can be used for distributed rate limiting.

---

# 76. Authorization

Every protected resource must verify ownership.

Example:

```text
GET /media/A
```

must verify:

```text
media.owner_id == authenticated_user.id
```

An ordinary user must never be able to retrieve another user's media by guessing a UUID.

---

# 77. Admin API

Administrative endpoints can be added under:

```text
/api/v1/admin
```

Examples:

```text
GET /api/v1/admin/workers
GET /api/v1/admin/jobs
GET /api/v1/admin/system
GET /api/v1/admin/audit-logs
```

All admin endpoints require:

```text
role = admin
```

---

# 78. Admin Worker Monitoring

```http
GET /api/v1/admin/workers
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "workerKey": "cpp-worker-01",
      "workerType": "cpp",
      "status": "online",
      "version": "1.0.0",
      "lastHeartbeatAt": "2026-09-07T18:35:00Z"
    }
  ]
}
```

---

# 79. API Security Rules

The API must:

* validate every request
* authenticate protected endpoints
* authorize resource ownership
* sanitize user-controlled metadata
* enforce upload size limits
* validate MIME types
* never trust file extensions
* never expose storage credentials
* never expose password hashes
* never expose session token hashes
* rate-limit authentication
* rate-limit expensive processing operations
* use HTTPS in production
* use parameterized SQL
* protect internal worker endpoints

---

# 80. Media Upload Security

The upload pipeline should validate:

```text
filename
content type
file size
extension
magic bytes
checksum
container format
codec
```

Example:

```text
demo.mp4
   │
   ▼
Declared MIME = video/mp4
   │
   ▼
Magic-byte validation
   │
   ▼
FFmpeg probe
   │
   ▼
Accepted
```

Never assume:

```text
filename = actual media type
```

---

# 81. API and Queue Boundary

The API should never perform heavy processing synchronously.

Bad:

```text
POST /media
      │
      ▼
FFmpeg
      │
      ▼
20-second request
```

Correct:

```text
POST /media
      │
      ▼
Create job
      │
      ▼
Redis
      │
      ▼
Worker
```

The API returns:

```http
202 Accepted
```

for asynchronous operations.

---

# 82. API and PostgreSQL Boundary

Node.js owns normal application database operations.

Workers may update processing state through:

```text
internal API
```

or a dedicated worker database-access layer.

The architecture should avoid giving arbitrary database permissions to worker processes.

---

# 83. API and Object Storage Boundary

The backend controls:

```text
bucket
object key
signed URL
expiration
permissions
```

The frontend receives only temporary access URLs.

Example:

```text
React
  │
  ▼
Node
  │
  ▼
Signed URL
  │
  ▼
Object Storage
```

---

# 84. API and Redis Boundary

Node.js publishes processing commands.

Example:

```json
{
  "jobId": "uuid",
  "type": "TRANSCRIBE",
  "mediaId": "uuid",
  "attempt": 1,
  "correlationId": "uuid"
}
```

Workers consume the message.

Redis should not be treated as the permanent database.

---

# 85. Queue Names

Recommended initial Redis streams/queues:

```text
media.jobs.cpp
media.jobs.python
media.events
media.dead-letter
```

Possible routing:

```text
MEDIA_PROBE
      │
      ▼
media.jobs.cpp

TRANSCRIBE
      │
      ▼
media.jobs.python
```

---

# 86. API-to-Worker Processing Example

```text
React
 │
 │ POST /media/123/jobs
 ▼
Node.js
 │
 ├── PostgreSQL
 │      │
 │      └── create job
 │
 └── Redis
        │
        ▼
    C++ Worker
        │
        ├── FFmpeg
        │
        ├── object storage
        │
        └── job progress
                │
                ▼
            Node.js
                │
                ▼
            PostgreSQL
                │
                ▼
            WebSocket
                │
                ▼
              React
```

---

# 87. API Contract Ownership

| Component      | Responsibility          |
| -------------- | ----------------------- |
| React          | API consumer            |
| Node.js        | Public API owner        |
| PostgreSQL     | Persistent state        |
| Redis          | Queue/event transport   |
| C++            | Native media processing |
| Python         | AI/ML processing        |
| Object Storage | Binary assets           |

---

# 88. TypeScript API Models

The Node.js project should maintain shared API types.

Recommended structure:

```text
gateway-node/
├── src/
│   ├── api/
│   │   ├── auth/
│   │   ├── media/
│   │   ├── jobs/
│   │   ├── transcripts/
│   │   ├── detections/
│   │   ├── scenes/
│   │   ├── clips/
│   │   ├── insights/
│   │   └── dashboard/
│   │
│   ├── middleware/
│   ├── services/
│   ├── repositories/
│   ├── queues/
│   ├── websocket/
│   └── types/
│
└── tests/
```

---

# 89. Validation

Use schema validation at the HTTP boundary.

Every request should follow:

```text
HTTP Request
     │
     ▼
Authentication
     │
     ▼
Authorization
     │
     ▼
Schema Validation
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

Invalid requests should never reach the repository layer.

---

# 90. API Service Architecture

Recommended Node.js structure:

```text
Request
   │
   ▼
Route
   │
   ▼
Controller
   │
   ▼
Validator
   │
   ▼
Service
   │
   ├── Repository
   │      │
   │      ▼
   │   PostgreSQL
   │
   └── Queue Service
          │
          ▼
        Redis
```

---

# 91. Repository Pattern

Repositories own database operations.

Example:

```text
MediaRepository
JobRepository
UserRepository
TranscriptRepository
DetectionRepository
ClipRepository
WorkerRepository
```

Services should not contain raw SQL scattered across controllers.

---

# 92. Service Layer

Example:

```text
MediaService
 ├── createUpload
 ├── completeUpload
 ├── getMedia
 ├── updateMedia
 └── deleteMedia

JobService
 ├── createJob
 ├── getJob
 ├── cancelJob
 ├── retryJob
 └── recoverJobs
```

---

# 93. API Observability

Every API request should log:

```text
requestId
userId
method
path
statusCode
durationMs
```

Example:

```json
{
  "level": "info",
  "requestId": "uuid",
  "userId": "uuid",
  "method": "POST",
  "path": "/api/v1/media",
  "statusCode": 201,
  "durationMs": 42
}
```

---

# 94. API Metrics

Initial metrics:

```text
http_requests_total
http_request_duration_ms
http_errors_total

media_uploads_total
media_processing_started_total
media_processing_completed_total

jobs_created_total
jobs_failed_total
jobs_retried_total
jobs_cancelled_total

websocket_connections
websocket_messages_total
```

---

# 95. API Performance Targets

Initial targets:

| Operation       |   Target |
| --------------- | -------: |
| Health check    |  < 20 ms |
| Authentication  | < 300 ms |
| Media metadata  | < 100 ms |
| Job creation    | < 150 ms |
| Job status      | < 100 ms |
| Job event query | < 200 ms |
| Dashboard       | < 300 ms |

These are initial engineering targets, not hard guarantees.

---

# 96. API Documentation Strategy

The implementation should generate OpenAPI documentation.

Recommended:

```text
docs/
└── openapi.yaml
```

The API specification should document:

* endpoints
* request schemas
* response schemas
* authentication
* errors
* query parameters
* WebSocket event contracts

The generated interactive API documentation can then be exposed in development.

---

# 97. OpenAPI Organization

Recommended sections:

```text
Authentication
Media
Assets
Jobs
Results
Transcripts
Detections
Scenes
Clips
AI Insights
Dashboard
Health
Admin
```

---

# 98. API Testing

Every endpoint should have:

```text
unit tests
integration tests
authorization tests
validation tests
error tests
```

Critical flows require end-to-end tests.

Example:

```text
Register
   ↓
Login
   ↓
Create media
   ↓
Upload
   ↓
Complete upload
   ↓
Create job
   ↓
Worker executes
   ↓
Result stored
   ↓
Frontend retrieves result
```

---

# 99. Complete MVP Endpoint List

```text
AUTH
────────────────────────────────────
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
GET    /api/v1/auth/me


MEDIA
────────────────────────────────────
POST   /api/v1/media
GET    /api/v1/media
GET    /api/v1/media/:mediaId
PATCH  /api/v1/media/:mediaId
DELETE /api/v1/media/:mediaId
POST   /api/v1/media/:mediaId/complete


ASSETS
────────────────────────────────────
GET    /api/v1/media/:mediaId/assets
GET    /api/v1/media/:mediaId/assets/:assetId
GET    /api/v1/media/:mediaId/assets/:assetId/url


JOBS
────────────────────────────────────
POST   /api/v1/media/:mediaId/jobs
GET    /api/v1/jobs
GET    /api/v1/jobs/:jobId
POST   /api/v1/jobs/:jobId/cancel
POST   /api/v1/jobs/:jobId/retry
GET    /api/v1/jobs/:jobId/events
GET    /api/v1/jobs/:jobId/result


RESULTS
────────────────────────────────────
GET    /api/v1/media/:mediaId/results


TRANSCRIPTS
────────────────────────────────────
GET    /api/v1/media/:mediaId/transcripts
GET    /api/v1/transcripts/:transcriptId
GET    /api/v1/transcripts/:transcriptId/segments


DETECTIONS
────────────────────────────────────
GET    /api/v1/media/:mediaId/detections
GET    /api/v1/media/:mediaId/detections/:detectionId


SCENES
────────────────────────────────────
GET    /api/v1/media/:mediaId/scenes
GET    /api/v1/media/:mediaId/scenes/:sceneId


CLIPS
────────────────────────────────────
POST   /api/v1/media/:mediaId/clips
GET    /api/v1/media/:mediaId/clips
GET    /api/v1/clips/:clipId
PATCH  /api/v1/clips/:clipId
DELETE /api/v1/clips/:clipId
POST   /api/v1/clips/:clipId/render


AI INSIGHTS
────────────────────────────────────
GET    /api/v1/media/:mediaId/insights
GET    /api/v1/media/:mediaId/insights/:insightId


SEARCH
────────────────────────────────────
GET    /api/v1/search


DASHBOARD
────────────────────────────────────
GET    /api/v1/dashboard


HEALTH
────────────────────────────────────
GET    /health
GET    /health/live
GET    /health/ready


INTERNAL WORKERS
────────────────────────────────────
POST   /internal/v1/workers/register
POST   /internal/v1/workers/:workerId/heartbeat
POST   /internal/v1/workers/:workerId/jobs/:jobId/start
POST   /internal/v1/workers/:workerId/jobs/:jobId/progress
POST   /internal/v1/workers/:workerId/jobs/:jobId/complete
POST   /internal/v1/workers/:workerId/jobs/:jobId/fail
```

---

# 100. Complete MVP Flow

The most important API workflow is:

```text
                 USER
                  │
                  ▼
          POST /auth/login
                  │
                  ▼
              JWT Token
                  │
                  ▼
       POST /media
                  │
                  ▼
       Upload URL returned
                  │
                  ▼
         Object Storage
                  │
                  ▼
    POST /media/:id/complete
                  │
                  ▼
       MEDIA_PROBE job
                  │
                  ▼
               Redis
                  │
                  ▼
            C++ Worker
                  │
                  ▼
       Probe + metadata
                  │
                  ▼
         PostgreSQL
                  │
                  ▼
        AUDIO_EXTRACT
                  │
                  ▼
             C++ Worker
                  │
                  ▼
           audio asset
                  │
                  ▼
            TRANSCRIBE
                  │
                  ▼
          Python Worker
                  │
                  ▼
             Transcript
                  │
                  ▼
          AI_ANALYSIS
                  │
                  ▼
          Python Worker
                  │
                  ▼
           AI Insights
                  │
                  ▼
          WebSocket Events
                  │
                  ▼
              React UI
```

---

# 101. Definition of Done

The API implementation is complete when:

* [ ] API versioning is implemented
* [ ] authentication works
* [ ] authorization works
* [ ] media CRUD works
* [ ] direct object-storage upload works
* [ ] upload completion works
* [ ] jobs can be created
* [ ] jobs can be queried
* [ ] jobs can be cancelled
* [ ] jobs can be retried
* [ ] job events are available
* [ ] processing results are available
* [ ] transcripts are available
* [ ] transcript segments are available
* [ ] detections are available
* [ ] scenes are available
* [ ] clips can be created
* [ ] clips can be rendered
* [ ] AI insights are available
* [ ] dashboard data is available
* [ ] worker registration works
* [ ] worker heartbeat works
* [ ] worker progress works
* [ ] worker completion works
* [ ] worker failure works
* [ ] WebSocket progress works
* [ ] request IDs are implemented
* [ ] correlation IDs are propagated
* [ ] idempotency is supported
* [ ] rate limiting is implemented
* [ ] validation is implemented
* [ ] OpenAPI documentation exists
* [ ] integration tests exist
* [ ] authorization tests exist
* [ ] error responses are standardized

---

# 102. Final API Architecture

```text
                         React
                           │
                    HTTPS / WebSocket
                           │
                           ▼
                 ┌──────────────────┐
                 │   Node Gateway   │
                 │   REST + WS      │
                 └────────┬─────────┘
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
     PostgreSQL         Redis        Object Storage
          │               │
          │               ├───────────────┐
          │               │               │
          │               ▼               ▼
          │          C++ Workers     Python Workers
          │               │               │
          └───────────────┴───────────────┘
                          │
                          ▼
                     Job Results
                          │
                          ▼
                     WebSocket
                          │
                          ▼
                        React
```

---

# 103. API Design Summary

The API follows these core rules:

```text
REST
    ↓
Public application interface

WebSocket
    ↓
Real-time updates

Redis
    ↓
Asynchronous job transport

PostgreSQL
    ↓
Persistent source of truth

Object Storage
    ↓
Binary media

C++ Workers
    ↓
High-performance media processing

Python Workers
    ↓
AI/ML processing
```

The API is therefore the **control-plane interface** of the entire platform.

The heavy media-processing data path never blocks normal HTTP requests.
