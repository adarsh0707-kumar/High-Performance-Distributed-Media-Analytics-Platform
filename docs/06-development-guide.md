# High-Performance Distributed Media Analytics Platform

# Development Guide

**Document:** `06-development-guide.md`
**Version:** 1.0
**Status:** Development Guide
**Author:** Adarsh Kumar
**Last Updated:** September 2026

---

# 1. Purpose

This document explains how to set up, develop, test, debug, and contribute to the High-Performance Distributed Media Analytics Platform.

It is intended for:

* project developers;
* contributors;
* backend engineers;
* C++ systems engineers;
* Python/AI engineers;
* frontend engineers;
* DevOps engineers;
* maintainers.

The guide assumes the project follows the architecture defined in:

```text
docs/
├── 01-product-requirements.md
├── 02-architecture.md
├── 03-data-model.md
├── 04-api-reference.md
└── 05-roadmap-and-phases.md
```

---

# 2. Development Philosophy

Development should follow:

```text
Small Change
    ↓
Build
    ↓
Unit Test
    ↓
Integration Test
    ↓
Review
    ↓
Commit
```

The system should remain runnable throughout development.

Avoid large changes that simultaneously modify:

```text
Node.js
C++
Python
React
Database
Infrastructure
```

unless the change genuinely requires cross-service coordination.

---

# 3. Technology Stack

## Frontend

```text
React
TypeScript
Vite
HTML5
CSS
WebSocket
```

Optional UI libraries may include:

```text
Lucide React
Recharts
React Router
```

---

## Backend

```text
Node.js
TypeScript
REST API
WebSocket
PostgreSQL client
Redis client
Object Storage SDK
```

---

## Native Media Engine

```text
C++
C++17
CMake
FFmpeg libraries
```

Primary FFmpeg components:

```text
libavformat
libavcodec
libavutil
libswscale
libswresample
libavfilter
```

---

## AI/ML Worker

```text
Python
Python 3.x
NumPy
ML/AI frameworks
Speech recognition
Computer vision
NLP
```

Specific models should remain behind interfaces so that they can be replaced without changing the worker architecture.

---

## Infrastructure

```text
PostgreSQL
Redis
S3-compatible Object Storage
Docker
Docker Compose
```

---

# 4. Prerequisites

A development machine should provide:

```text
Git
Docker
Docker Compose
Node.js
npm
Python
pip
CMake
GCC/G++
FFmpeg development libraries
pkg-config
```

Verify:

```bash
git --version
docker --version
docker compose version
node --version
npm --version
python3 --version
cmake --version
gcc --version
g++ --version
ffmpeg -version
pkg-config --version
```

---

# 5. Linux Development Setup

The recommended development environment is Linux.

For Arch/Garuda-based systems:

```bash
sudo pacman -Syu
```

Install core development packages:

```bash
sudo pacman -S \
    git \
    base-devel \
    cmake \
    pkgconf \
    ffmpeg \
    docker \
    docker-compose \
    nodejs \
    npm \
    python \
    python-pip
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
docker run --rm hello-world
```

If Docker requires elevated privileges during development, configure the Docker group according to the local system's security policy.

---

# 6. Repository Setup

Clone the repository:

```bash
git clone <repository-url>
cd High-Performance-Distributed-Media-Analytics-Platform
```

Create the development branch:

```bash
git checkout -b develop
```

Create an environment file:

```bash
cp .env.example .env
```

Never commit `.env`.

---

# 7. Repository Structure

The recommended project structure is:

```text
High-Performance-Distributed-Media-Analytics-Platform/
│
├── docs/
│   ├── decisions/
│   ├── diagrams/
│   ├── 01-product-requirements.md
│   ├── 02-architecture.md
│   ├── 03-data-model.md
│   ├── 04-api-reference.md
│   ├── 05-roadmap-and-phases.md
│   ├── 06-development-guide.md
│   ├── 07-security.md
│   ├── 08-gap-analysis.md
│   ├── 09-testing-strategy.md
│   ├── 10-glossary.md
│   └── README.md
│
├── gateway-node/
│   ├── src/
│   ├── tests/
│   ├── package.json
│   ├── tsconfig.json
│   └── README.md
│
├── media-engine-cpp/
│   ├── include/
│   ├── src/
│   ├── tests/
│   ├── CMakeLists.txt
│   └── README.md
│
├── analytics-python/
│   ├── app/
│   ├── tests/
│   ├── pyproject.toml
│   ├── requirements.txt
│   └── README.md
│
├── frontend-react/
│   ├── src/
│   ├── public/
│   ├── tests/
│   ├── package.json
│   └── README.md
│
├── infra/
│   ├── docker/
│   ├── postgres/
│   ├── redis/
│   └── object-storage/
│
├── scripts/
├── tests/
│
├── .github/
│   └── workflows/
│
├── docker-compose.yml
├── Makefile
├── .env.example
├── .gitignore
└── README.md
```

---

# 8. Environment Configuration

All environment-specific configuration should come from environment variables.

Example:

```env
APP_ENV=development
APP_NAME=media-analytics-platform

NODE_ENV=development
API_PORT=8080

DATABASE_URL=postgresql://media:media@localhost:5432/media_analytics

REDIS_URL=redis://localhost:6379

OBJECT_STORAGE_ENDPOINT=http://localhost:9000
OBJECT_STORAGE_REGION=us-east-1
OBJECT_STORAGE_BUCKET=media
OBJECT_STORAGE_ACCESS_KEY=development
OBJECT_STORAGE_SECRET_KEY=development

JWT_ACCESS_SECRET=change-me
JWT_REFRESH_SECRET=change-me

LOG_LEVEL=debug

CORS_ORIGIN=http://localhost:5173
```

The actual production secrets must never be placed in source control.

---

# 9. Environment Separation

Three primary environments are recommended.

```text
development
     │
     ▼
staging
     │
     ▼
production
```

## Development

Characteristics:

```text
Local services
Debug logging
Test credentials
Developer-friendly configuration
```

## Staging

Characteristics:

```text
Production-like configuration
Realistic workloads
Security testing
Integration testing
```

## Production

Characteristics:

```text
Restricted credentials
Structured logging
Monitoring
Alerting
Backups
Strict security
```

---

# 10. Root Makefile

The project should provide common commands through a root Makefile.

Example interface:

```bash
make help
make setup
make build
make test
make lint
make format
make dev
make docker-up
make docker-down
make clean
```

Recommended command structure:

```text
make
 ├── setup
 ├── build
 ├── test
 ├── lint
 ├── format
 ├── dev
 ├── docker-up
 ├── docker-down
 └── clean
```

The Makefile acts as a consistent developer interface across the polyglot repository.

---

# 11. Node.js Development

Navigate to the gateway:

```bash
cd gateway-node
```

Install dependencies:

```bash
npm install
```

Development server:

```bash
npm run dev
```

Build:

```bash
npm run build
```

Run production build:

```bash
npm start
```

Tests:

```bash
npm test
```

Lint:

```bash
npm run lint
```

Formatting:

```bash
npm run format
```

---

# 12. Node.js Architecture

The backend should use layered architecture.

```text
HTTP Request
     ↓
Controller
     ↓
Service
     ↓
Repository
     ↓
PostgreSQL
```

For asynchronous operations:

```text
Controller
     ↓
Service
     ↓
Job Publisher
     ↓
Redis
```

Recommended source structure:

```text
gateway-node/src/
├── config/
├── controllers/
├── database/
├── errors/
├── middleware/
├── models/
├── repositories/
├── routes/
├── schemas/
├── services/
├── workers/
├── websocket/
├── logging/
├── utils/
└── server.ts
```

---

# 13. Node.js Coding Rules

Use TypeScript instead of JavaScript for application code.

Prefer:

```ts
interface MediaService
{
    ...
}
```

over untyped objects.

Avoid:

```ts
const data: any = ...
```

unless there is a documented reason.

Use explicit error handling:

```text
Request
  ↓
Validation
  ↓
Service
  ↓
Repository
  ↓
Error Mapping
  ↓
HTTP Response
```

Controllers should remain thin.

Business logic belongs in services.

Database queries belong in repositories.

---

# 14. PostgreSQL Development

Start PostgreSQL:

```bash
docker compose up -d postgres
```

Check:

```bash
docker compose ps
```

Connect:

```bash
psql "$DATABASE_URL"
```

List tables:

```sql
\dt
```

Inspect schema:

```sql
\d media
\d jobs
\d processing_results
```

---

# 15. Database Migrations

Database changes must be version controlled.

Do not modify production tables manually.

Recommended migration structure:

```text
infra/postgres/
├── migrations/
│   ├── 001_initial_schema.sql
│   ├── 002_add_job_events.sql
│   ├── 003_add_processing_manifests.sql
│   └── ...
└── seed/
```

Each migration should be:

```text
Ordered
Repeatable
Reviewable
Testable
```

Example:

```text
001_initial_schema.sql
002_add_media_indexes.sql
003_add_worker_capabilities.sql
```

---

# 16. Database Rules

Use PostgreSQL as the source of truth for persistent state.

Redis must not become the permanent database for:

```text
Users
Media metadata
Job history
Processing results
Audit logs
```

Redis is primarily used for:

```text
Queueing
Transient coordination
Event transport
Short-lived state
```

---

# 17. Redis Development

Start Redis:

```bash
docker compose up -d redis
```

Check:

```bash
docker compose ps redis
```

Test connectivity:

```bash
redis-cli ping
```

Expected:

```text
PONG
```

Inspect queues using the appropriate Redis tooling for the selected queue implementation.

---

# 18. Job Development

A job should follow:

```text
CREATE
  ↓
QUEUED
  ↓
LEASED
  ↓
RUNNING
  ↓
COMPLETED
```

Failure:

```text
RUNNING
   ↓
FAILED
   ↓
RETRYING
   ↓
QUEUED
```

Permanent failure:

```text
FAILED
  ↓
DEAD-LETTER
```

---

# 19. Job Implementation Rules

Every job should contain enough information for a worker to execute it independently.

Conceptually:

```json
{
  "jobId": "uuid",
  "type": "MEDIA_PROBE",
  "mediaId": "uuid",
  "attempt": 1,
  "correlationId": "uuid",
  "idempotencyKey": "unique-key",
  "input": {},
  "createdAt": "timestamp"
}
```

Workers should not depend on browser state.

Workers should not assume that an API process remains alive.

---

# 20. Worker Lease Model

A worker should claim a job for a limited period.

```text
Job
 ↓
LEASED
 ↓
Worker executes
 ↓
Heartbeat
 ↓
Complete
```

If the worker disappears:

```text
Lease expires
     ↓
Recovery process
     ↓
Job becomes retryable
```

This prevents permanently stuck jobs.

---

# 21. C++ Development

Navigate to the C++ project:

```bash
cd media-engine-cpp
```

Configure:

```bash
cmake -S . -B build
```

Build:

```bash
cmake --build build -j
```

Run tests:

```bash
ctest --test-dir build --output-on-failure
```

Clean build:

```bash
rm -rf build
cmake -S . -B build
cmake --build build -j
```

---

# 22. C++ Build Configuration

The project should use:

```text
C++17
```

Recommended compiler flags:

```text
-Wall
-Wextra
-Wpedantic
-Wconversion
-Wshadow
```

Warnings should be treated seriously.

Release builds should use optimization appropriate for benchmarked workloads.

---

# 23. C++ Architecture

Recommended structure:

```text
media-engine-cpp/
├── include/
│   ├── core/
│   ├── media/
│   ├── ffmpeg/
│   ├── processing/
│   ├── queue/
│   └── utils/
│
├── src/
│   ├── core/
│   ├── media/
│   ├── ffmpeg/
│   ├── processing/
│   ├── queue/
│   └── utils/
│
├── tests/
└── CMakeLists.txt
```

Dependency direction:

```text
Application
    ↓
Processing
    ↓
Media
    ↓
FFmpeg Adapter
    ↓
FFmpeg
```

Avoid allowing FFmpeg-specific types to leak unnecessarily throughout the entire codebase.

---

# 24. FFmpeg Development

Verify FFmpeg:

```bash
ffmpeg -version
```

Check development libraries:

```bash
pkg-config --modversion libavformat
pkg-config --modversion libavcodec
pkg-config --modversion libavutil
```

Check compiler flags:

```bash
pkg-config --cflags --libs \
    libavformat \
    libavcodec \
    libavutil \
    libswscale \
    libswresample
```

The C++ project should prefer native FFmpeg APIs rather than invoking the `ffmpeg` command-line executable as its primary processing mechanism.

---

# 25. C++ Memory Rules

Media processing can involve very large buffers.

Never assume:

```text
Input file size ≈ memory required
```

Instead:

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
```

Use bounded processing.

Avoid loading entire videos into memory.

Every FFmpeg resource must have a clearly defined ownership lifecycle.

---

# 26. C++ Error Handling

Errors should be converted into application-level errors.

Example categories:

```text
MEDIA_NOT_FOUND
UNSUPPORTED_FORMAT
CODEC_ERROR
DECODE_ERROR
ENCODE_ERROR
INVALID_TIME_RANGE
IO_ERROR
MEMORY_ERROR
FFMPEG_ERROR
CANCELLED
TIMEOUT
```

Do not expose raw FFmpeg errors directly to users without contextual mapping.

---

# 27. Python Development

Navigate:

```bash
cd analytics-python
```

Create a virtual environment:

```bash
python3 -m venv .venv
```

Activate:

```bash
source .venv/bin/activate
```

Upgrade tooling:

```bash
python -m pip install --upgrade pip
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Run tests:

```bash
pytest
```

Run lint:

```bash
ruff check .
```

Format:

```bash
ruff format .
```

Deactivate:

```bash
deactivate
```

---

# 28. Python Architecture

Recommended:

```text
analytics-python/
├── app/
│   ├── config/
│   ├── queue/
│   ├── storage/
│   ├── manifest/
│   ├── transcription/
│   ├── vision/
│   ├── nlp/
│   ├── models/
│   ├── persistence/
│   ├── worker/
│   └── main.py
│
├── tests/
├── pyproject.toml
└── requirements.txt
```

The AI layer should use interfaces around models.

Example:

```text
TranscriptionService
       │
       ├── LocalModel
       ├── CloudModel
       └── MockModel
```

This makes testing possible without requiring large models.

---

# 29. Python Model Versioning

Every AI result should record enough information to reproduce or understand the result.

Example:

```json
{
  "model": "transcription-model",
  "modelVersion": "1.0.0",
  "language": "en",
  "confidence": 0.94
}
```

Models should not be silently upgraded in production.

---

# 30. Frontend Development

Navigate:

```bash
cd frontend-react
```

Install:

```bash
npm install
```

Run:

```bash
npm run dev
```

Build:

```bash
npm run build
```

Preview:

```bash
npm run preview
```

Test:

```bash
npm test
```

Lint:

```bash
npm run lint
```

---

# 31. React Architecture

Recommended:

```text
frontend-react/src/
├── app/
├── components/
├── features/
│   ├── auth/
│   ├── media/
│   ├── jobs/
│   ├── transcripts/
│   ├── detections/
│   ├── insights/
│   └── dashboard/
├── hooks/
├── layouts/
├── pages/
├── services/
├── stores/
├── types/
├── utils/
└── main.tsx
```

Organize code by feature where practical.

---

# 32. Frontend State

Separate:

```text
Server State
```

from:

```text
UI State
```

Server state includes:

```text
Media
Jobs
Transcripts
Detections
Insights
```

UI state includes:

```text
Selected timeline position
Open panel
Theme
Modal visibility
Current tool
```

Avoid putting every state variable into one global store.

---

# 33. Media Studio Development

The primary interface should support:

```text
Media Preview
Timeline
Processing Status
Transcript
Detections
AI Insights
Clips
```

Conceptually:

```text
┌───────────────────────────────────────┐
│ Toolbar                               │
├───────────────────────────────────────┤
│                                       │
│             Media Player              │
│                                       │
├───────────────────────────────────────┤
│ Timeline                              │
├───────────────────────────────────────┤
│ Transcript │ Detection │ AI Insights  │
└───────────────────────────────────────┘
```

---

# 34. API Client

The frontend should not directly construct arbitrary HTTP requests throughout components.

Use a centralized API layer:

```text
services/
├── apiClient.ts
├── authApi.ts
├── mediaApi.ts
├── jobsApi.ts
├── transcriptApi.ts
├── detectionApi.ts
└── insightApi.ts
```

This provides:

```text
Authentication
Error handling
Request IDs
API versioning
Response parsing
```

---

# 35. WebSocket Development

WebSocket flow:

```text
React
  ↓
WebSocket
  ↓
Node.js
  ↓
Event Source
```

Client should handle:

```text
CONNECTING
CONNECTED
RECONNECTING
DISCONNECTED
```

Events should be typed.

Example:

```ts
type JobProgressEvent =
{
    type: "job.progress";
    jobId: string;
    progress: number;
    stage: string;
};
```

Do not use untyped event payloads throughout the frontend.

---

# 36. Object Storage Development

Local development should use an S3-compatible storage implementation.

Conceptual layout:

```text
bucket/
├── original/
├── proxy/
├── audio/
├── thumbnails/
├── clips/
└── manifests/
```

Object keys should not depend directly on user-provided filenames.

Prefer:

```text
media/{mediaId}/original/{assetId}
```

instead of:

```text
uploads/my-video-final-final.mp4
```

---

# 37. Media Artifact Rules

Each generated artifact should have metadata:

```text
Asset ID
Media ID
Asset Type
Object Key
Content Type
Size
Checksum
Created At
Producer
Producer Version
```

Example:

```json
{
  "type": "thumbnail",
  "objectKey": "media/uuid/thumbnails/frame-001.jpg",
  "producer": "cpp-media-engine",
  "producerVersion": "0.1.0"
}
```

---

# 38. Processing Manifest

The manifest is the contract between processing stages.

Example:

```json
{
  "schemaVersion": "1.0",
  "mediaId": "uuid",
  "source": {
    "assetId": "uuid",
    "objectKey": "media/uuid/original/source.mp4"
  },
  "artifacts": [
    {
      "type": "audio",
      "objectKey": "media/uuid/audio/main.wav"
    }
  ]
}
```

The manifest should be versioned.

Workers should reject unsupported manifest versions cleanly.

---

# 39. Local Development Modes

Two development modes are recommended.

## Mode A — Native Services

Run directly:

```text
Node
C++
Python
React
```

while infrastructure runs in Docker:

```text
PostgreSQL
Redis
Object Storage
```

This is preferred for active development.

---

## Mode B — Full Docker

Run:

```bash
docker compose up --build
```

This verifies service packaging and networking.

---

# 40. Recommended Daily Development Workflow

Start infrastructure:

```bash
docker compose up -d postgres redis object-storage
```

Start backend:

```bash
cd gateway-node
npm run dev
```

Start C++ worker:

```bash
cd media-engine-cpp
./build/media-worker
```

Start Python worker:

```bash
cd analytics-python
source .venv/bin/activate
python -m app.main
```

Start frontend:

```bash
cd frontend-react
npm run dev
```

System:

```text
Browser
   │
   ▼
React
   │
   ▼
Node.js
   │
 ┌─┴───────────────┐
 ▼                 ▼
PostgreSQL        Redis
                    │
             ┌──────┴──────┐
             ▼             ▼
            C++          Python
             │             │
             └──────┬──────┘
                    ▼
              Object Storage
```

---

# 41. Health Checks

Every service should expose or provide an appropriate health mechanism.

## API

```bash
curl http://localhost:8080/health
```

## PostgreSQL

```bash
pg_isready
```

## Redis

```bash
redis-cli ping
```

## Object Storage

Verify through the configured storage health mechanism.

## C++ Worker

Worker heartbeat:

```text
worker.online
worker.heartbeat
worker.offline
```

## Python Worker

Same worker lifecycle model.

---

# 42. Logging

All services should produce structured logs.

Example:

```json
{
  "timestamp": "2026-09-07T12:00:00Z",
  "level": "INFO",
  "service": "cpp-worker",
  "message": "Job completed",
  "jobId": "uuid",
  "mediaId": "uuid",
  "correlationId": "uuid"
}
```

Avoid:

```text
Processing failed
```

Prefer:

```text
Job processing failed
jobId=...
mediaId=...
stage=...
errorCode=...
```

---

# 43. Correlation IDs

Every user-triggered processing operation should have a correlation ID.

```text
HTTP Request
     │
     ▼
Node.js
     │
     ├── PostgreSQL
     │
     ├── Redis
     │
     └── Worker
             │
             ▼
          Python
```

The same correlation ID should be available throughout the operation.

This enables:

```text
Request
 ↓
Job
 ↓
Worker
 ↓
AI processing
 ↓
Result
```

to be traced logically.

---

# 44. Debugging Workflow

When something fails:

## Step 1 — Identify service

```text
Frontend?
Node?
PostgreSQL?
Redis?
C++?
Python?
Storage?
```

## Step 2 — Check health

```bash
docker compose ps
```

## Step 3 — Check logs

```bash
docker compose logs -f <service>
```

## Step 4 — Check request ID

Find the corresponding:

```text
requestId
correlationId
jobId
mediaId
```

## Step 5 — Inspect database

```sql
SELECT *
FROM jobs
WHERE id = '...';
```

## Step 6 — Inspect queue

Check whether the job is:

```text
Queued
Leased
Running
Stuck
Retrying
Dead-lettered
```

---

# 45. Common C++ Debugging Tools

Use:

```text
gdb
valgrind
AddressSanitizer
UndefinedBehaviorSanitizer
```

Example:

```bash
cmake -S . -B build \
    -DCMAKE_BUILD_TYPE=Debug \
    -DENABLE_ASAN=ON
```

Run:

```bash
gdb ./build/media-worker
```

Memory testing:

```bash
valgrind --leak-check=full ./build/media-worker
```

These tools should be part of development and debugging rather than only used after crashes occur.

---

# 46. Python Debugging

Useful tools:

```text
pytest
pdb
logging
ruff
mypy
```

Example:

```bash
python -m pdb -m app.main
```

Run targeted test:

```bash
pytest tests/test_manifest.py -v
```

---

# 47. Node.js Debugging

Use:

```bash
node --inspect
```

or the IDE debugger.

Log:

```text
requestId
userId
mediaId
jobId
```

Never log:

```text
passwords
access tokens
refresh tokens
storage credentials
private keys
```

---

# 48. Frontend Debugging

Use browser developer tools for:

```text
Console
Network
WebSocket
Application Storage
Performance
```

Check:

```text
HTTP status
Request payload
Response payload
WebSocket connection
WebSocket events
Authentication state
```

---

# 49. Testing Workflow

Before creating a pull request:

```bash
make build
make test
make lint
```

If formatting is part of CI:

```bash
make format
```

Run the end-to-end test suite when changing:

```text
API contracts
Job lifecycle
Worker contracts
Database schema
Media processing
WebSocket events
```

---

# 50. Unit Testing

Unit tests should isolate individual components.

Examples:

```text
Node:
  AuthService
  MediaService
  JobService

C++:
  TimeRange
  MediaProbe
  ClipCalculator
  ManifestWriter

Python:
  ManifestParser
  TranscriptProcessor
  DetectionProcessor

React:
  UploadComponent
  JobProgress
  Timeline
```

---

# 51. Integration Testing

Integration tests should verify real service boundaries.

Example:

```text
Node
 ↓
PostgreSQL
```

and:

```text
Node
 ↓
Redis
 ↓
Worker
```

and:

```text
Worker
 ↓
Object Storage
```

Use isolated test infrastructure whenever possible.

---

# 52. End-to-End Test

The primary E2E test is:

```text
Register
  ↓
Login
  ↓
Upload
  ↓
Create Job
  ↓
Queue
  ↓
C++ Worker
  ↓
Artifact
  ↓
Python Worker
  ↓
Result
  ↓
React
```

Expected:

```text
HTTP 2xx
Job COMPLETED
Artifacts exist
Results exist
UI displays result
```

---

# 53. Test Media Dataset

Maintain a controlled test dataset.

Example:

```text
tests/media/
├── tiny-video.mp4
├── short-video.mp4
├── audio.wav
├── multi-track.mp4
├── invalid-media.bin
└── long-video.mp4
```

Do not commit extremely large media files to Git.

Use:

```text
Git LFS
Object Storage
CI artifact storage
```

when appropriate.

---

# 54. Performance Testing

Performance tests should be repeatable.

Example:

```text
Dataset A
100 MB
10 minutes

Dataset B
500 MB
30 minutes

Dataset C
1 GB
60 minutes
```

Measure:

```text
Upload
Probe
Decode
Transcode
Audio extraction
Transcription
Detection
AI processing
Total duration
Memory
CPU
Queue latency
```

Record results in:

```text
docs/benchmarks/
```

---

# 55. Benchmark Rules

A benchmark should specify:

```text
Hardware
OS
Compiler
Build type
FFmpeg version
Model version
Input media
Concurrency
Result
```

Example:

```text
CPU:
8-core

RAM:
32 GB

Build:
Release

Input:
1 GB MP4

Workers:
4

Result:
...
```

Without environment information, benchmark results are difficult to compare.

---

# 56. Git Workflow

Create a feature branch:

```bash
git checkout develop
git pull --rebase
git checkout -b feature/media-upload
```

Develop:

```text
Edit
 ↓
Build
 ↓
Test
 ↓
Commit
```

Commit:

```bash
git add .
git commit -m "feat(media): implement media upload workflow"
```

Push:

```bash
git push -u origin feature/media-upload
```

Open a pull request.

---

# 57. Commit Convention

Use:

```text
feat:
fix:
refactor:
perf:
test:
docs:
build:
ci:
chore:
```

Examples:

```text
feat(auth): implement refresh token rotation
feat(queue): add job leasing
feat(media): implement object storage upload
feat(cpp): add FFmpeg media probing

fix(queue): recover expired job leases
fix(api): reject unauthorized media access

perf(cpp): reduce frame allocation overhead

test(worker): add retry recovery tests

docs(api): document job endpoints
```

---

# 58. Pull Request Rules

A pull request should contain:

```text
Purpose
Changes
Testing
Potential Risks
Screenshots when UI changes
Migration notes when DB changes
```

Example:

```text
## Summary

Implemented media upload through object storage.

## Changes

- Added media creation endpoint
- Added object storage service
- Added upload completion endpoint
- Added checksum validation

## Testing

- Unit tests
- Integration tests
- Manual upload test

## Migration

Added media_assets fields.
```

---

# 59. Database Change PR Rules

Any schema change must include:

```text
Migration
Rollback consideration
Index impact
Data migration
Application compatibility
```

Avoid:

```text
Application deployed first
Database changed later
```

unless the migration is explicitly designed for backward compatibility.

---

# 60. API Contract Rules

API changes should be backwards compatible whenever practical.

For breaking changes:

```text
/api/v1
```

should remain stable while a future:

```text
/api/v2
```

can be introduced.

Changes to:

```text
Request schema
Response schema
Job message
Manifest schema
WebSocket events
```

must be documented.

---

# 61. Worker Contract Rules

Workers must treat incoming jobs as untrusted input.

Validate:

```text
Job ID
Job type
Media ID
Manifest version
Object key
Parameters
Time ranges
```

Workers should never blindly trust data from Redis.

---

# 62. Graceful Shutdown

Every service should support graceful shutdown.

Conceptually:

```text
SIGTERM
  ↓
Stop accepting new work
  ↓
Finish current safe operation
  ↓
Release resources
  ↓
Close DB
  ↓
Close Redis
  ↓
Close storage clients
  ↓
Exit
```

Workers should not abandon actively leased jobs without attempting appropriate state recovery.

---

# 63. Worker Startup

Worker startup should follow:

```text
Process starts
    ↓
Load configuration
    ↓
Initialize logging
    ↓
Connect to Redis
    ↓
Connect to storage
    ↓
Register worker
    ↓
Send heartbeat
    ↓
Consume jobs
```

If a critical dependency cannot be initialized, the worker should fail clearly rather than silently operating in a broken state.

---

# 64. Worker Capability Registration

Workers should report capabilities.

Example:

```json
{
  "workerType": "cpp-media",
  "version": "0.1.0",
  "capabilities": [
    "MEDIA_PROBE",
    "THUMBNAIL_GENERATE",
    "AUDIO_EXTRACT",
    "VIDEO_CLIP"
  ]
}
```

Python worker:

```json
{
  "workerType": "python-analytics",
  "version": "0.1.0",
  "capabilities": [
    "TRANSCRIPTION",
    "DETECTION",
    "AI_ANALYSIS"
  ]
}
```

This allows future capability-aware scheduling.

---

# 65. Configuration Rules

Configuration should be centralized.

Avoid hardcoding:

```text
Database URL
Redis URL
Storage credentials
JWT secrets
Model paths
Worker concurrency
```

Use environment variables or configuration files.

---

# 66. Secrets Management

Never commit:

```text
.env
private keys
JWT secrets
cloud credentials
database passwords
API keys
```

Use:

```text
.env.example
```

for documentation.

Example:

```env
DATABASE_URL=
REDIS_URL=
OBJECT_STORAGE_ACCESS_KEY=
OBJECT_STORAGE_SECRET_KEY=
JWT_ACCESS_SECRET=
```

---

# 67. Docker Development

Build everything:

```bash
docker compose build
```

Start:

```bash
docker compose up
```

Background:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Stop and remove volumes:

```bash
docker compose down -v
```

Use the last command carefully because it destroys local persistent data.

---

# 68. Docker Logs

All services:

```bash
docker compose logs -f
```

Specific service:

```bash
docker compose logs -f gateway
```

Worker:

```bash
docker compose logs -f cpp-worker
```

Python:

```bash
docker compose logs -f python-worker
```

---

# 69. Docker Health

Inspect:

```bash
docker compose ps
```

A healthy deployment should show:

```text
postgres       healthy
redis          healthy
object-storage healthy
gateway        healthy
cpp-worker     running
python-worker  running
frontend       running
```

---

# 70. Clean Development Environment

If dependencies become inconsistent:

```bash
docker compose down
```

Then rebuild:

```bash
docker compose build --no-cache
docker compose up
```

For Node:

```bash
rm -rf node_modules
npm install
```

For C++:

```bash
rm -rf build
cmake -S . -B build
cmake --build build
```

For Python:

```bash
rm -rf .venv
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Do not use destructive cleanup commands without checking what data they remove.

---

# 71. Development Troubleshooting

## PostgreSQL connection failure

Check:

```bash
docker compose ps postgres
```

Then:

```bash
docker compose logs postgres
```

Check:

```text
DATABASE_URL
Port
Username
Password
Database name
```

---

## Redis connection failure

Check:

```bash
redis-cli ping
```

Expected:

```text
PONG
```

Verify:

```text
REDIS_URL
Port
Container
Network
```

---

## FFmpeg compile failure

Check:

```bash
pkg-config --modversion libavcodec
pkg-config --modversion libavformat
```

If unavailable, install the FFmpeg development package appropriate for the OS.

---

## C++ runtime crash

Run:

```bash
gdb ./build/media-worker
```

Then inspect the backtrace:

```text
bt
```

For memory issues:

```bash
valgrind --leak-check=full ./build/media-worker
```

---

## Python dependency failure

Recreate the environment:

```bash
rm -rf .venv
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

---

## Frontend cannot connect to API

Check:

```text
API URL
CORS
API port
Browser Network tab
Backend health endpoint
```

---

# 72. Development Security Checklist

Before submitting a security-sensitive change:

```text
[ ] Input validation
[ ] Authorization
[ ] Authentication
[ ] Secrets handling
[ ] Logging safety
[ ] File path safety
[ ] Upload validation
[ ] Resource limits
[ ] Error handling
```

Never trust:

```text
Filename
MIME type
User ID
Media ID
Job ID
Object key
Worker message
```

without validation.

---

# 73. Resource Limits

Media processing is resource-intensive.

Configure limits for:

```text
Maximum upload size
Maximum processing duration
Maximum concurrent jobs
Maximum worker memory
Maximum CPU usage
Maximum decoded frame dimensions
Maximum audio duration
```

The exact values should be benchmarked and configured per environment.

---

# 74. Local Development Security

Even local development should avoid:

```text
Hardcoded production credentials
Real customer media
Production databases
Production JWT keys
```

Use synthetic/test media and development credentials.

---

# 75. Documentation Workflow

Documentation is part of implementation.

When changing:

```text
Architecture
API
Database
Security
Worker contracts
Deployment
```

update the corresponding document in the same pull request.

Relevant documentation:

```text
Architecture change
→ 02-architecture.md

Database change
→ 03-data-model.md

API change
→ 04-api-reference.md

Roadmap change
→ 05-roadmap-and-phases.md

Development workflow
→ 06-development-guide.md

Security change
→ 07-security.md
```

---

# 76. Architecture Decision Records

Significant technical decisions should receive an ADR.

Example:

```text
docs/decisions/
├── ADR-001-polyglot-architecture.md
├── ADR-002-ffmpeg-native-integration.md
├── ADR-003-redis-job-queue.md
├── ADR-004-object-storage.md
├── ADR-005-postgresql.md
└── ADR-006-worker-isolation.md
```

Use ADRs when introducing:

```text
New infrastructure
Major framework
Database architecture
Processing model
Queue technology
Security architecture
Deployment model
```

---

# 77. Local Service Ports

Recommended development ports:

| Service                |   Port |
| ---------------------- | -----: |
| React                  | `5173` |
| Node.js API            | `8080` |
| PostgreSQL             | `5432` |
| Redis                  | `6379` |
| Object Storage API     | `9000` |
| Object Storage Console | `9001` |

These values may be changed through configuration.

---

# 78. Service Startup Order

Infrastructure:

```text
PostgreSQL
Redis
Object Storage
```

then:

```text
Node.js
```

then:

```text
C++ Worker
Python Worker
```

and finally:

```text
React
```

However, production orchestration must use health/readiness checks rather than assuming startup order alone guarantees dependency availability.

---

# 79. Development Workflow by Feature

For a new feature:

## Step 1

Update requirements if necessary.

## Step 2

Update architecture if necessary.

## Step 3

Update database model if necessary.

## Step 4

Update API contract.

## Step 5

Implement backend.

## Step 6

Implement worker functionality.

## Step 7

Implement frontend.

## Step 8

Add tests.

## Step 9

Add observability.

## Step 10

Update documentation.

## Step 11

Benchmark if performance-sensitive.

---

# 80. Example Feature Workflow — Video Clip

A new clip feature should proceed as:

```text
Requirement
    ↓
API contract
    ↓
Database model
    ↓
POST /media/:id/clips
    ↓
VIDEO_CLIP job
    ↓
Redis
    ↓
C++ worker
    ↓
FFmpeg
    ↓
Clip artifact
    ↓
media_assets
    ↓
processing_results
    ↓
WebSocket
    ↓
React
```

Testing should cover the entire path.

---

# 81. Example Feature Workflow — Transcription

```text
Audio extraction
      ↓
Audio artifact
      ↓
TRANSCRIPTION job
      ↓
Python worker
      ↓
Speech model
      ↓
Transcript
      ↓
Transcript segments
      ↓
AI/NLP
      ↓
Insights
      ↓
React
```

The Python worker should not directly manipulate frontend state.

---

# 82. CI Pipeline

Recommended GitHub Actions workflows:

```text
.github/workflows/
├── build.yml
├── test.yml
├── lint.yml
├── docker.yml
└── security.yml
```

## Build

Validate:

```text
Node build
C++ build
Python package
React build
```

## Test

Run:

```text
Unit tests
Integration tests
```

## Lint

Run:

```text
TypeScript lint
C++ static analysis
Python lint
Frontend lint
```

## Docker

Validate:

```text
Dockerfiles
Docker Compose
Container startup
Health checks
```

## Security

Run:

```text
Dependency scanning
Secret scanning
Container scanning
Static analysis
```

---

# 83. Pre-Commit Checklist

Before committing:

```text
[ ] Code compiles
[ ] Tests pass
[ ] Lint passes
[ ] Formatting passes
[ ] No secrets
[ ] Logs are safe
[ ] Documentation updated
[ ] Migration included if needed
```

---

# 84. Pull Request Checklist

```text
## Implementation

[ ] Feature complete
[ ] Error handling implemented
[ ] Logging implemented
[ ] Metrics considered

## Testing

[ ] Unit tests
[ ] Integration tests
[ ] E2E tests if required

## Security

[ ] Authentication
[ ] Authorization
[ ] Input validation
[ ] Secret handling

## Documentation

[ ] API updated
[ ] Architecture updated
[ ] README updated if needed

## Deployment

[ ] Docker updated
[ ] Environment variables documented
```

---

# 85. Production-Like Local Test

Before release, run the entire platform using Docker:

```bash
docker compose down -v
docker compose build
docker compose up -d
```

Verify:

```bash
docker compose ps
```

Then:

```text
Register
 ↓
Login
 ↓
Upload test media
 ↓
Start processing
 ↓
Observe queue
 ↓
Observe workers
 ↓
Wait for completion
 ↓
Verify artifacts
 ↓
Verify database
 ↓
Verify UI
```

---

# 86. Release Candidate Test

Run:

```bash
make clean
make build
make test
make lint
docker compose build
docker compose up -d
```

Then execute:

```text
Smoke tests
Integration tests
E2E tests
Performance tests
Failure recovery tests
Security tests
```

Record results before tagging a release.

---

# 87. Recommended Release Flow

```text
feature branch
      ↓
pull request
      ↓
CI
      ↓
code review
      ↓
develop
      ↓
staging
      ↓
release candidate
      ↓
main
      ↓
tag
```

Example:

```bash
git tag v0.1.0
git push origin v0.1.0
```

---

# 88. Versioning

Use:

```text
MAJOR.MINOR.PATCH
```

Examples:

```text
v0.1.0
v0.2.0
v0.2.1
v1.0.0
```

Before 1.0, APIs may evolve more rapidly.

After 1.0, breaking API changes require explicit versioning and migration documentation.

---

# 89. Development Environment Definition

A developer environment is considered ready when:

```text
[✓] Git works
[✓] Docker works
[✓] PostgreSQL works
[✓] Redis works
[✓] Object Storage works
[✓] Node.js builds
[✓] C++ builds
[✓] Python worker starts
[✓] React starts
[✓] API health works
[✓] Database migrations work
[✓] Test suite executes
```

---

# 90. First Development Session

A new developer should be able to perform:

```bash
git clone <repository-url>

cd High-Performance-Distributed-Media-Analytics-Platform

cp .env.example .env

docker compose up -d postgres redis object-storage

cd gateway-node
npm install
npm run dev
```

In another terminal:

```bash
cd media-engine-cpp

cmake -S . -B build

cmake --build build
```

Another:

```bash
cd analytics-python

python3 -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt

python -m app.main
```

Another:

```bash
cd frontend-react

npm install
npm run dev
```

Then open the React development URL.

---

# 91. Development Golden Path

The recommended developer experience is:

```text
Clone
  ↓
Configure .env
  ↓
Start infrastructure
  ↓
Run migrations
  ↓
Start API
  ↓
Start C++ worker
  ↓
Start Python worker
  ↓
Start React
  ↓
Upload test media
  ↓
Process
  ↓
Inspect results
```

A developer should not need to understand the entire distributed architecture before successfully running the first end-to-end workflow.

---

# 92. Engineering Principles

The project follows these engineering principles:

## Principle 1 — Explicit Boundaries

Each service should have a clear responsibility.

## Principle 2 — Persistent Truth

PostgreSQL remains the source of truth for persistent application state.

## Principle 3 — Disposable Workers

Workers can be stopped and restarted without destroying the system.

## Principle 4 — Idempotency

Retrying work should not corrupt results.

## Principle 5 — Observable Systems

Important operations must be measurable and traceable.

## Principle 6 — Secure by Default

Input should be considered untrusted.

## Principle 7 — Test Before Optimize

Performance changes require benchmarks.

## Principle 8 — Documentation Is Code

Architectural and API changes require documentation updates.

---

# 93. Definition of Developer Readiness

The development environment is considered successfully configured when a developer can execute:

```text
Upload
   ↓
Queue
   ↓
C++
   ↓
Artifact
   ↓
Python
   ↓
Result
   ↓
React
```

without manually modifying source code between stages.

This is the most important local development acceptance test.

---

# 94. Final Development Checklist

```text
ENVIRONMENT
[ ] Git
[ ] Docker
[ ] Node.js
[ ] Python
[ ] C++
[ ] CMake
[ ] FFmpeg

INFRASTRUCTURE
[ ] PostgreSQL
[ ] Redis
[ ] Object Storage

BACKEND
[ ] API
[ ] Authentication
[ ] Database
[ ] Job system
[ ] WebSocket

C++
[ ] FFmpeg
[ ] Probe
[ ] Decode
[ ] Audio
[ ] Thumbnail
[ ] Clip

PYTHON
[ ] Worker
[ ] Manifest
[ ] Transcription
[ ] Detection
[ ] AI

FRONTEND
[ ] Authentication
[ ] Upload
[ ] Media Library
[ ] Media Studio
[ ] Timeline
[ ] Results
[ ] Realtime

QUALITY
[ ] Unit tests
[ ] Integration tests
[ ] E2E tests
[ ] Benchmarks
[ ] Security checks
[ ] CI

DOCUMENTATION
[ ] Architecture
[ ] API
[ ] Database
[ ] ADRs
[ ] Development guide
```

---

# 95. Final Developer Workflow

The complete development lifecycle is:

```text
                ┌─────────────────┐
                │     IDEA        │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │   REQUIREMENT   │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │    DESIGN       │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │ IMPLEMENTATION  │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │     TEST        │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │   BENCHMARK     │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │ SECURITY REVIEW │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │ DOCUMENTATION   │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │   CODE REVIEW   │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │      MERGE      │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │    RELEASE      │
                └─────────────────┘
```

---

# 96. Conclusion

The High-Performance Distributed Media Analytics Platform is intentionally developed as a **multi-service systems project**, not as a single monolithic application.

The development model separates:

```text
React
  ↓
User Experience

Node.js
  ↓
Control Plane

PostgreSQL
  ↓
Persistent State

Redis
  ↓
Job Coordination

Object Storage
  ↓
Binary Assets

C++
  ↓
High-Performance Media Processing

Python
  ↓
AI/ML Analytics
```

The most important development rule is to maintain a working vertical slice throughout the project:

```text
Upload
  ↓
Store
  ↓
Queue
  ↓
Process
  ↓
Analyze
  ↓
Persist
  ↓
Display
```

Every major feature should strengthen this pipeline without creating unnecessary coupling between services.

Once this development workflow is established, the project can progress from local development to Docker Compose, staging, cloud deployment, and eventually independently scalable worker infrastructure without requiring a fundamental rewrite of the core architecture.
