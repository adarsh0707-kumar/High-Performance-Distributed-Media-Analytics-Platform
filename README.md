## High-Performance Distributed Media Analytics Platform

> A production-oriented, polyglot distributed media processing platform for high-performance video/audio analysis, intelligent metadata extraction, automated clipping, transcription, and AI-powered media insights.

[![Architecture](https://img.shields.io/badge/architecture-distributed-blue)](#architecture)
[![C++](https://img.shields.io/badge/C%2B%2B-17-blue)](#media-engine)
[![Python](https://img.shields.io/badge/Python-3.x-yellow)](#analytics-worker)
[![Node.js](https://img.shields.io/badge/Node.js-TypeScript-green)](#gateway)
[![React](https://img.shields.io/badge/React-TypeScript-61DAFB)](#frontend)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-database-336791)](#data-layer)
[![Redis](https://img.shields.io/badge/Redis-queue-red)](#job-processing)
[![Docker](https://img.shields.io/badge/Docker-containerized-2496ED)](#deployment)

---

## 1. Overview

The **High-Performance Distributed Media Analytics Platform** is a multi-service system designed to process large and complex video/audio assets through a distributed processing pipeline.

The platform combines multiple technologies according to their strengths:

* **C/C++** for performance-critical media processing
* **FFmpeg native libraries** for decoding, encoding, demuxing, muxing, filtering, audio processing, and media inspection
* **Python** for analytics, AI/ML pipelines, transcription, detection, and intelligent processing
* **Node.js + TypeScript** for the API gateway, authentication, orchestration, job management, and real-time communication
* **React + TypeScript** for the Media Studio frontend
* **PostgreSQL** for persistent metadata and system state
* **Redis** for asynchronous job coordination
* **Object Storage** for large media files and generated artifacts
* **Docker** for reproducible deployment and service isolation

The architecture separates the **control plane** from the **data plane**, allowing computationally expensive media workloads to scale independently from the API and frontend.

---

## 2. Problem Statement

Modern media applications frequently need to perform computationally expensive operations such as:

* Video decoding
* Audio extraction
* Thumbnail generation
* Video clipping
* Scene detection
* Face detection
* Object detection
* Speech transcription
* Sentiment analysis
* AI-generated summaries
* Media metadata extraction

Performing all of these operations synchronously inside a traditional web backend creates several problems:

* High API latency
* CPU and memory contention
* Poor scalability
* Worker crashes affecting the API
* Difficult retry and recovery mechanisms
* Large media files consuming application-server resources
* Limited observability
* Tight coupling between application logic and media-processing code

This project addresses these problems through a distributed asynchronous processing architecture.

---

## 3. Goals

### Primary Goals

* Build a production-oriented distributed media processing architecture.
* Separate API/control-plane workloads from media-processing workloads.
* Process media using native C/C++ and FFmpeg libraries.
* Support Python-based AI/ML analytics.
* Implement reliable asynchronous job processing.
* Store large binary objects outside PostgreSQL.
* Provide real-time processing progress.
* Support independent horizontal scaling of workers.
* Provide strong authentication and authorization.
* Build reproducible local and containerized environments.
* Establish measurable performance and reliability targets.

### Secondary Goals

* Demonstrate systems programming skills.
* Demonstrate distributed-systems design.
* Demonstrate C++/Python/TypeScript interoperability.
* Demonstrate production-oriented API design.
* Demonstrate database and queue architecture.
* Demonstrate testing, observability, security, and deployment practices.

---

## 4. Non-Goals

The initial MVP does **not** attempt to provide:

* Full professional collaborative video editing
* Multi-region deployment
* Kubernetes orchestration
* GPU cluster scheduling
* Real-money media processing workloads
* Enterprise identity federation
* Real-time collaborative editing
* Large-scale model training

These capabilities can be introduced in later phases.

---

# 5. Architecture

The platform is divided into two major architectural domains.

```text
                    ┌──────────────────────────┐
                    │       React + TS         │
                    │      Media Studio        │
                    └────────────┬─────────────┘
                                 │
                                 │ HTTPS / WebSocket
                                 ▼
                    ┌──────────────────────────┐
                    │   Node.js + TypeScript   │
                    │       API Gateway        │
                    │                          │
                    │ Auth / API / Jobs / WS   │
                    └──────┬───────┬───────┬───┘
                           │       │       │
                           ▼       ▼       ▼
                    PostgreSQL   Redis   Object Storage
                           │       │       │
                           │       │       │
                    ┌──────┘       │       └──────────┐
                    │              │                  │
                    ▼              ▼                  ▼
              Metadata DB     Job Queue        Media / Artifacts
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                         ▼                     ▼
                  ┌─────────────┐       ┌─────────────┐
                  │ C++ Workers │       │Python Worker│
                  │             │       │             │
                  │ FFmpeg      │       │ AI / ML     │
                  │ Media       │       │ Analytics   │
                  │ Processing  │       │             │
                  └─────────────┘       └─────────────┘
```

---

## 6. Control Plane

The control plane manages the lifecycle of the platform.

### Components

* React frontend
* Node.js API
* PostgreSQL
* Redis
* Authentication
* Job orchestration
* Worker registration
* Real-time events
* API authorization
* Metadata management

The control plane should remain lightweight and responsive even while workers process large media files.

---

# 7. Data Plane

The data plane performs computationally expensive processing.

### Components

* C++ media workers
* FFmpeg native libraries
* Python analytics workers
* AI/ML providers
* Temporary processing storage
* Object-storage artifacts

Workers are designed to be:

* Disposable
* Independently scalable
* Capability-aware
* Recoverable
* Observable
* Resource-limited
* Idempotent

---

# 8. End-to-End Data Flow

```text
User
 │
 ▼
React Media Studio
 │
 │ Upload
 ▼
Node.js API
 │
 ├──────────────► PostgreSQL
 │                    │
 │                    └── Media Metadata
 │
 └──────────────► Object Storage
                      │
                      └── Original Media
                              │
                              ▼
                         Create Job
                              │
                              ▼
                           Redis
                              │
                              ▼
                      C++ Media Worker
                              │
                              ▼
                       FFmpeg Processing
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
             Metadata     Thumbnail      Clip
                 │            │            │
                 └────────────┼────────────┘
                              ▼
                       Processing Manifest
                              │
                              ▼
                       Python Worker
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
              Transcription Detection AI Insights
                    │         │         │
                    └─────────┼─────────┘
                              ▼
                         PostgreSQL
                              │
                              ▼
                       WebSocket Event
                              │
                              ▼
                       React Media Studio
```

---

# 9. Processing Pipeline

The MVP processing pipeline is:

```text
Upload
  ↓
Media Registration
  ↓
Object Storage
  ↓
Job Creation
  ↓
Redis Queue
  ↓
C++ Worker
  ↓
Media Probe
  ↓
Decode / Extract
  ↓
Thumbnail / Clip / Audio
  ↓
Processing Manifest
  ↓
Python Analytics
  ↓
Transcription / Detection / AI
  ↓
Persist Results
  ↓
WebSocket Progress
  ↓
React Media Studio
```

---

# 10. Media Engine

The C++ media engine is responsible for performance-critical operations.

## Responsibilities

* Media probing
* Container inspection
* Stream discovery
* Video decoding
* Audio decoding
* Thumbnail generation
* Clip extraction
* Audio extraction
* Audio resampling
* Audio downmixing
* Frame processing
* Metadata extraction
* Artifact generation

The engine integrates with FFmpeg through its native libraries rather than treating FFmpeg primarily as a command-line subprocess.

### Planned FFmpeg libraries

```text
libavformat
libavcodec
libavutil
libswscale
libswresample
libavfilter
```

### Design Principles

* RAII resource management
* Explicit ownership
* Bounded memory usage
* No uncontrolled shell execution
* Structured error handling
* Deterministic processing
* Safe cleanup
* Performance instrumentation
* Sanitizer-friendly implementation

---

# 11. Analytics Worker

The Python worker handles higher-level analytics and AI/ML processing.

## Responsibilities

* Speech transcription
* Transcript segmentation
* Speaker-related analysis
* Sentiment analysis
* Object detection
* Face detection
* Scene analysis
* Image recognition
* AI-generated insights
* Result normalization

Python receives processing artifacts and manifests from the C++ layer rather than depending on unsafe direct shared memory.

Example:

```text
C++ Worker
    │
    ├── audio.wav
    ├── thumbnail.jpg
    ├── clip.mp4
    └── manifest.json
            │
            ▼
      Object Storage
            │
            ▼
     Python Worker
            │
            ├── transcription
            ├── detection
            └── AI analysis
            │
            ▼
        Results
```

---

# 12. Job Processing

Processing is asynchronous.

A job follows a lifecycle similar to:

```text
CREATED
   │
   ▼
QUEUED
   │
   ▼
LEASED
   │
   ▼
RUNNING
   │
   ├───────────────► COMPLETED
   │
   ├───────────────► FAILED
   │
   └───────────────► CANCELLED
                         │
                         ▼
                       RETRY
                         │
                         ▼
                       QUEUED
```

Each job supports:

* Priority
* Attempts
* Maximum retry count
* Backoff
* Lease expiration
* Worker ownership
* Progress updates
* Cancellation
* Idempotency
* Error classification
* Recovery

---

# 13. Job Queue

Redis is initially used as the asynchronous coordination layer.

The database remains the **source of truth** for durable job state.

```text
PostgreSQL
    │
    │ durable state
    ▼
Job State

Redis
    │
    │ transport / dispatch
    ▼
Worker Queue
```

This separation prevents Redis from becoming the authoritative system-of-record database.

---

# 14. Storage Architecture

Large media objects should not be stored directly inside PostgreSQL.

### PostgreSQL

Stores:

* Users
* Media metadata
* Jobs
* Job attempts
* Processing state
* Transcripts
* Detections
* Scenes
* Clips metadata
* AI insights
* Audit records
* Worker state

### Object Storage

Stores:

* Original video
* Original audio
* Extracted audio
* Thumbnails
* Generated clips
* Intermediate artifacts
* Processing manifests
* AI artifacts

Example:

```text
Object Storage
│
├── media/
│   └── <media-id>/
│       ├── original/
│       ├── thumbnails/
│       ├── clips/
│       ├── audio/
│       └── artifacts/
│
└── jobs/
    └── <job-id>/
        ├── manifest.json
        └── intermediate/
```

---

# 15. Repository Structure

```text
High-Performance-Distributed-Media-Analytics-Platform/
│
├── docs/
│   ├── decisions/
│   │   ├── ADR-001-polyglot-architecture.md
│   │   ├── ADR-002-ffmpeg-native-integration.md
│   │   ├── ADR-003-redis-job-queue.md
│   │   ├── ADR-004-object-storage.md
│   │   ├── ADR-005-postgresql.md
│   │   └── ADR-006-worker-isolation.md
│   │
│   ├── diagrams/
│   │   ├── system-architecture.png
│   │   ├── data-flow.png
│   │   ├── job-lifecycle.png
│   │   ├── media-processing-pipeline.png
│   │   └── deployment-architecture.png
│   │
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
│   ├── src/
│   ├── include/
│   ├── tests/
│   ├── CMakeLists.txt
│   └── README.md
│
├── analytics-python/
│   ├── src/
│   ├── tests/
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
│
├── tests/
│   ├── integration/
│   ├── e2e/
│   ├── performance/
│   └── fixtures/
│
├── .github/
│   └── workflows/
│       ├── build.yml
│       ├── test.yml
│       ├── lint.yml
│       ├── docker.yml
│       └── security.yml
│
├── docker-compose.yml
├── Makefile
├── .env.example
├── .gitignore
└── README.md
```

---

# 16. Technology Stack

| Layer            | Technology                                | Purpose                       |
| ---------------- | ----------------------------------------- | ----------------------------- |
| Frontend         | React + TypeScript                        | Media Studio                  |
| API              | Node.js + TypeScript                      | Gateway/control plane         |
| Media Engine     | C++17                                     | High-performance processing   |
| Media Framework  | FFmpeg                                    | Native media processing       |
| Analytics        | Python                                    | AI/ML and analytics           |
| Database         | PostgreSQL                                | Durable metadata/state        |
| Queue            | Redis                                     | Asynchronous job coordination |
| Object Storage   | S3-compatible                             | Media/artifacts               |
| Realtime         | WebSocket                                 | Progress/events               |
| Containerization | Docker                                    | Deployment/isolation          |
| Build            | CMake                                     | C++ build system              |
| Testing          | Vitest / GoogleTest / pytest / Playwright | Automated testing             |
| Load Testing     | k6                                        | Performance testing           |

---

# 17. API

The API is versioned under:

```text
/api/v1
```

Major API areas include:

```text
/api/v1/auth
/api/v1/media
/api/v1/jobs
/api/v1/transcripts
/api/v1/detections
/api/v1/scenes
/api/v1/clips
/api/v1/insights
/api/v1/search
/api/v1/dashboard
```

Health endpoints:

```text
GET /health
GET /health/live
GET /health/ready
```

WebSocket:

```text
/ws
```

Example real-time event:

```json
{
  "type": "job.progress",
  "jobId": "uuid",
  "progress": 67,
  "stage": "transcription",
  "timestamp": "2026-09-07T12:00:00Z"
}
```

---

# 18. Security

Security is treated as an architectural requirement rather than a final deployment task.

Core principles include:

* Authentication for protected resources
* Authorization on every resource
* Object-level access control
* Parameterized SQL
* Strict request validation
* Secure media upload validation
* Generated object-storage keys
* Private object storage
* Private PostgreSQL and Redis
* Worker authentication
* Worker capability validation
* Resource limits
* Processing timeouts
* Retry limits
* Sandboxed native processing
* No unsafe shell execution
* Secret isolation
* TLS in production
* Audit logging
* Dependency scanning
* Container scanning
* Malformed-media testing

Uploaded media must always be treated as **untrusted input**.

---

# 19. Observability

Every significant operation should be traceable through structured metadata.

The platform uses:

* Request IDs
* Correlation IDs
* Job IDs
* Worker IDs
* Structured logs
* Metrics
* Processing duration
* Queue latency
* Worker utilization
* Error rates
* Retry counts
* Artifact generation metrics

Example:

```text
Request
  │
  └── correlation_id
        │
        ├── job_id
        │     │
        │     ├── worker_id
        │     ├── attempt_id
        │     └── processing stage
        │
        └── API logs
```

---

# 20. Performance Targets

Initial engineering targets:

| Metric                    |                       Target |
| ------------------------- | ---------------------------: |
| API health response       |                     < 100 ms |
| Job creation              |                     < 200 ms |
| Queue dispatch            |                     < 500 ms |
| Media metadata extraction | < 5 s for typical test media |
| Worker startup            |                        < 3 s |
| WebSocket progress delay  |                        < 1 s |
| API availability target   |                        99.9% |
| Job state durability      |                         100% |
| Retry-safe processing     |                     Required |
| Idempotent job execution  |                     Required |

These values are engineering targets and should be validated through benchmark tests rather than assumed.

---

# 21. Development Environment

Recommended development environment:

```text
Linux
Git
CMake
GCC / Clang
FFmpeg
Node.js
npm
Python
PostgreSQL
Redis
Docker
Docker Compose
```

### Verify dependencies

```bash
git --version
cmake --version
gcc --version
ffmpeg -version
node --version
npm --version
python3 --version
docker --version
docker compose version
psql --version
redis-cli --version
```

---

# 22. Quick Start

Clone the repository:

```bash
git clone <repository-url>
cd High-Performance-Distributed-Media-Analytics-Platform
```

Create environment configuration:

```bash
cp .env.example .env
```

Start infrastructure:

```bash
docker compose up -d postgres redis object-storage
```

Check services:

```bash
docker compose ps
```

---

## Start Node.js Gateway

```bash
cd gateway-node

npm install
npm run dev
```

Build:

```bash
npm run build
```

Test:

```bash
npm test
```

---

## Build C++ Media Engine

```bash
cd media-engine-cpp

cmake -S . -B build
cmake --build build -j
```

Run tests:

```bash
ctest --test-dir build --output-on-failure
```

Verify FFmpeg development libraries:

```bash
pkg-config --modversion libavformat
pkg-config --modversion libavcodec
pkg-config --modversion libavutil
```

---

## Start Python Worker

```bash
cd analytics-python

python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
pip install -r requirements.txt

pytest
```

Lint:

```bash
ruff check .
ruff format .
```

---

## Start React Frontend

```bash
cd frontend-react

npm install
npm run dev
```

The development frontend runs on:

```text
http://localhost:5173
```

---

# 23. Docker Development

Build all services:

```bash
docker compose build
```

Start:

```bash
docker compose up -d
```

View logs:

```bash
docker compose logs -f
```

Check status:

```bash
docker compose ps
```

Stop:

```bash
docker compose down
```

Remove development volumes:

```bash
docker compose down -v
```

---

# 24. Root Makefile

The root Makefile provides a unified developer interface.

Typical commands:

```bash
make build
make test
make lint
make clean
```

Infrastructure:

```bash
make up
make down
make logs
```

A developer should not need to remember every service-specific command for common workflows.

---

# 25. Testing Strategy

The project follows a layered testing pyramid.

```text
                 ┌──────────────┐
                 │   E2E Tests  │
                 └──────────────┘
               ┌──────────────────┐
               │ Integration Tests│
               └──────────────────┘
          ┌────────────────────────────┐
          │        Unit Tests          │
          └────────────────────────────┘
```

Testing technologies:

### C++

* GoogleTest
* AddressSanitizer
* UndefinedBehaviorSanitizer
* ThreadSanitizer
* libFuzzer

### Python

* pytest
* Hypothesis
* Ruff

### Node.js

* Vitest
* Supertest

### React

* Vitest
* React Testing Library
* Playwright

### Infrastructure

* Disposable PostgreSQL
* Redis
* S3-compatible object storage
* Integration test containers

### Performance

* k6
* C++ benchmarks
* Processing corpus
* CPU/memory measurements

---

# 26. Reliability

The platform is designed around failure as an expected condition.

Workers may:

* Crash
* Timeout
* Lose network connectivity
* Become unhealthy
* Exceed resource limits
* Produce invalid artifacts
* Lose leases

The system therefore implements:

* Heartbeats
* Leases
* Retry policies
* Exponential backoff
* Idempotency
* Attempt tracking
* Failure classification
* Recovery
* Dead-letter handling
* Graceful shutdown

---

# 27. Development Roadmap

The implementation follows a vertical-slice strategy.

```text
Repository Foundation
        ↓
Development Tooling
        ↓
PostgreSQL
        ↓
Node Gateway
        ↓
Authentication
        ↓
Object Storage
        ↓
Media Upload
        ↓
Redis
        ↓
Job Lifecycle
        ↓
Worker Framework
        ↓
C++ FFmpeg Engine
        ↓
Python Analytics
        ↓
Manifest Contract
        ↓
End-to-End Pipeline
        ↓
React Media Studio
        ↓
WebSocket
        ↓
Dashboard
        ↓
Search
        ↓
Testing
        ↓
Security
        ↓
Docker
        ↓
Production Readiness
```

---

# 28. Current MVP

The MVP focuses on one complete processing path:

```text
Upload
  ↓
Store
  ↓
Create Job
  ↓
Queue
  ↓
C++ Worker
  ↓
FFmpeg Probe
  ↓
Generate Metadata
  ↓
Generate Thumbnail
  ↓
Create Manifest
  ↓
Persist Result
  ↓
Display in React
```

Once this vertical slice is stable, additional analytics can be introduced.

---

# 29. Future Capabilities

Potential future versions may add:

### AI

* Advanced speech-to-text
* Speaker diarization
* Emotion analysis
* Content moderation
* Semantic search
* Multimodal embeddings
* AI-generated summaries
* Automatic highlight generation

### Media

* GPU-accelerated decoding
* GPU inference
* Advanced scene detection
* Automatic clip generation
* Multi-track audio processing
* Transcoding profiles

### Distributed Infrastructure

* Kubernetes
* GPU worker pools
* Autoscaling
* Multi-region processing
* Distributed tracing
* Dedicated worker pools
* Advanced scheduling

### Product

* Timeline editor
* Collaborative editing
* Team workspaces
* Media collections
* Search across transcripts
* AI-assisted editing
* Workflow templates

---

# 30. Architecture Decision Records

Important architectural decisions are documented in:

```text
docs/decisions/
```

Current ADRs:

```text
ADR-001-polyglot-architecture.md
ADR-002-ffmpeg-native-integration.md
ADR-003-redis-job-queue.md
ADR-004-object-storage.md
ADR-005-postgresql.md
ADR-006-worker-isolation.md
```

The purpose of ADRs is to record **why** an architectural decision was made, not merely what was implemented.

---

# 31. Documentation

The project documentation is organized as follows:

| Document                     | Purpose                  |
| ---------------------------- | ------------------------ |
| `01-product-requirements.md` | Product requirements     |
| `02-architecture.md`         | System architecture      |
| `03-data-model.md`           | PostgreSQL data model    |
| `04-api-reference.md`        | API contract             |
| `05-roadmap-and-phases.md`   | Development roadmap      |
| `06-development-guide.md`    | Developer handbook       |
| `07-security.md`             | Security architecture    |
| `08-gap-analysis.md`         | Implementation readiness |
| `09-testing-strategy.md`     | Testing architecture     |
| `10-glossary.md`             | Canonical terminology    |

---

# 32. Engineering Principles

The project follows these principles:

### 1. Performance where it matters

Use C++ for CPU-intensive media operations instead of forcing every workload through the application layer.

### 2. Use the right language for the right problem

```text
C++       → Media processing
Python    → AI / ML / analytics
TypeScript→ API / orchestration
React     → User interface
SQL       → Durable state
Redis     → Job coordination
```

### 3. Asynchronous by default

Large media processing should never block API requests.

### 4. Database as source of truth

Redis transports work; PostgreSQL records durable state.

### 5. Workers must be disposable

A worker failure must not corrupt the control plane.

### 6. Treat media as hostile

Uploaded files are untrusted input.

### 7. Measure performance

Performance claims must be supported by benchmarks.

### 8. Design for recovery

Every long-running job needs a recovery strategy.

### 9. Prefer explicit contracts

Service boundaries should use versioned, validated contracts.

### 10. Document architectural decisions

Important tradeoffs belong in ADRs.

---

# 33. Project Status

### Architecture

**Status:** Defined

* [x] Product requirements
* [x] System architecture
* [x] Data model
* [x] API design
* [x] Security architecture
* [x] Testing strategy
* [x] Roadmap
* [x] Gap analysis
* [x] Glossary
* [x] ADR structure

### Implementation

**Status:** In active development

The project should be considered **implementation-ready but not production-ready** until the core vertical slice, automated testing, security hardening, observability, and deployment validation are complete.

---

# 34. Definition of Done

A feature is considered complete when:

* [ ] Requirements are defined
* [ ] Architecture impact is understood
* [ ] Implementation is complete
* [ ] Unit tests exist
* [ ] Integration tests exist where required
* [ ] Error handling is implemented
* [ ] Logging is implemented
* [ ] Metrics are available where appropriate
* [ ] Security implications are reviewed
* [ ] Performance impact is measured where relevant
* [ ] Documentation is updated
* [ ] CI passes
* [ ] Code review is complete

---

# 35. License

Add the project's selected license here before public release.

Example:

```text
MIT License
```

---

# 36. Author

**Adarsh Kumar**

This project is designed as a portfolio-grade demonstration of:

* Distributed systems
* Systems programming
* C/C++ media processing
* FFmpeg integration
* Python AI/ML pipelines
* Backend engineering
* React application development
* Database architecture
* Asynchronous job processing
* Security engineering
* Automated testing
* Performance engineering
* Containerized deployment

---

# 37. Final Architecture

The intended production evolution is:

```text
                    ┌──────────────────────┐
                    │    Media Studio      │
                    │    React + TypeScript │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │     API Gateway      │
                    │  Node.js + TypeScript│
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
        ┌──────────┐     ┌──────────┐    ┌──────────────┐
        │PostgreSQL│     │  Redis   │    │Object Storage│
        └──────────┘     └────┬─────┘    └──────┬───────┘
                              │                 │
                     ┌────────┴────────┐        │
                     │                 │        │
                     ▼                 ▼        │
              ┌────────────┐    ┌────────────┐ │
              │C++ Workers │    │Python      │ │
              │            │    │Workers     │ │
              │FFmpeg      │    │AI / ML     │ │
              │Media       │    │Analytics   │ │
              │Engine      │    │            │ │
              └─────┬──────┘    └─────┬──────┘ │
                    │                  │        │
                    └──────────────────┼────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │ Processing Results│
                              │ Metadata / AI     │
                              │ Insights / Clips  │
                              └────────┬─────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │   WebSocket      │
                              │ Realtime Updates  │
                              └────────┬─────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │   Media Studio   │
                              └──────────────────┘
```

**The central architectural principle is simple:**

> **Keep the control plane responsive, move expensive work to isolated workers, store binaries in object storage, keep durable state in PostgreSQL, and use explicit contracts between processing stages.**

---


## Licence

[MIT](./LICENSE).

---

If this project was useful to you, consider [![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-support-yellow?logo=buy-me-a-coffee\&logoColor=white)](https://buymeacoffee.com/adarsh12kumar)
