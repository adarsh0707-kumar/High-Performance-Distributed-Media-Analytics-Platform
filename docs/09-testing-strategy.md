# High-Performance Distributed Media Analytics Platform

# Testing Strategy

**Document:** `09-testing-strategy.md`
**Version:** 1.0
**Status:** Implementation Baseline
**Author:** Adarsh Kumar
**Project:** High-Performance Distributed Media Analytics Platform

---

## 1. Purpose

This document defines the complete testing strategy for the High-Performance Distributed Media Analytics Platform.

The platform is a polyglot distributed system containing:

* React + TypeScript frontend
* Node.js + TypeScript API gateway
* PostgreSQL database
* Redis job queue
* Object storage
* C++ media-processing workers
* FFmpeg native libraries
* Python analytics and AI/ML workers
* WebSocket real-time communication
* Docker-based infrastructure

Testing must therefore validate not only individual functions but also:

* service boundaries
* API contracts
* database consistency
* queue semantics
* worker leases
* retries
* media-processing correctness
* artifact integrity
* AI pipeline behavior
* concurrency
* failure recovery
* security
* performance
* end-to-end user workflows

The primary objective is:

> Every production-critical behavior must have an automated test at the lowest practical testing level.

---

# 2. Testing Philosophy

The project follows these principles:

1. **Test behavior, not implementation details.**
2. **Prefer fast tests close to the code.**
3. **Use integration tests for infrastructure boundaries.**
4. **Use E2E tests for critical business workflows.**
5. **Treat uploaded media as untrusted test input.**
6. **Test failure paths as seriously as success paths.**
7. **Test concurrency explicitly.**
8. **Make distributed operations deterministic wherever possible.**
9. **Use reproducible test datasets.**
10. **Performance-test the C++ media engine independently.**
11. **Never rely exclusively on code coverage.**
12. **Block releases on critical security or correctness failures.**

---

# 3. Testing Objectives

The testing program must verify:

### Functional correctness

* authentication works
* media can be registered
* media can be uploaded
* jobs can be created
* workers can consume jobs
* C++ processing produces valid artifacts
* Python processing consumes valid artifacts
* results are persisted
* frontend displays results

### Reliability

* jobs survive worker failure
* expired leases can be recovered
* retries behave correctly
* duplicate messages do not duplicate work
* cancellation works
* partial failures are handled correctly

### Security

* unauthorized resources cannot be accessed
* malicious media cannot bypass validation
* SQL injection is prevented
* path traversal is prevented
* internal worker endpoints are protected
* object storage remains private

### Performance

* media processing meets target throughput
* API latency remains acceptable
* queue throughput scales
* database queries remain efficient
* WebSocket updates remain responsive

### Maintainability

* contracts are tested
* regressions are detected automatically
* test failures are diagnosable
* tests run consistently in local development and CI

---

# 4. Testing Pyramid

The project follows a modified testing pyramid.

```text
                         ┌───────────────────┐
                         │    Manual / UX    │
                         │     Validation    │
                         └─────────┬─────────┘
                                   │
                         ┌─────────▼─────────┐
                         │      E2E Tests    │
                         │ Critical Workflows│
                         └─────────┬─────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │     Integration Tests       │
                    │ DB / Redis / Storage / API  │
                    └──────────────┬──────────────┘
                                   │
             ┌─────────────────────▼─────────────────────┐
             │                Contract Tests              │
             │ API / Worker / Manifest / WebSocket       │
             └─────────────────────┬─────────────────────┘
                                   │
        ┌──────────────────────────▼──────────────────────────┐
        │                    Unit Tests                        │
        │ C++ / Python / Node.js / React / Utility Functions  │
        └──────────────────────────────────────────────────────┘
```

Target distribution:

| Test Type   |     Target Share |
| ----------- | ---------------: |
| Unit        |           60–70% |
| Contract    |           10–15% |
| Integration |           15–20% |
| E2E         |            5–10% |
| Manual      | Exploratory only |

The percentages are guidelines rather than hard requirements.

---

# 5. Test Architecture

Recommended repository structure:

```text
tests/
├── fixtures/
│   ├── media/
│   │   ├── video-small.mp4
│   │   ├── video-h264.mp4
│   │   ├── video-audio.mp4
│   │   ├── audio.wav
│   │   ├── image.jpg
│   │   ├── malformed.mp4
│   │   └── oversized.metadata.json
│   │
│   ├── manifests/
│   │   ├── valid-v1.json
│   │   ├── invalid-v1.json
│   │   └── unsupported-version.json
│   │
│   └── api/
│       ├── users.json
│       └── media.json
│
├── unit/
│   ├── gateway/
│   ├── cpp/
│   ├── python/
│   └── frontend/
│
├── integration/
│   ├── database/
│   ├── redis/
│   ├── object-storage/
│   ├── gateway/
│   ├── workers/
│   └── pipeline/
│
├── contract/
│   ├── api/
│   ├── worker/
│   ├── manifest/
│   └── websocket/
│
├── e2e/
│   ├── auth/
│   ├── media/
│   ├── processing/
│   └── dashboard/
│
├── performance/
│   ├── api/
│   ├── cpp/
│   ├── python/
│   ├── queue/
│   └── end-to-end/
│
├── security/
│   ├── api/
│   ├── upload/
│   ├── auth/
│   ├── workers/
│   └── storage/
│
└── reports/
```

---

# 6. Testing Environments

The project uses four primary environments.

## 6.1 Local Unit Environment

Used for:

* unit tests
* static analysis
* formatting
* fast contract tests

External infrastructure should be mocked where practical.

---

## 6.2 Integration Environment

Uses disposable infrastructure:

```text
Docker
├── PostgreSQL
├── Redis
├── Object Storage
├── Node Gateway
├── C++ Worker
└── Python Worker
```

This environment validates service interactions.

---

## 6.3 CI Environment

CI should run:

```text
Lint
  ↓
Unit Tests
  ↓
Contract Tests
  ↓
Integration Tests
  ↓
Security Tests
  ↓
Build
  ↓
E2E
  ↓
Performance Smoke Tests
```

---

## 6.4 Production-like Environment

Used before release candidates.

It should resemble the production topology:

```text
React
   │
   ▼
Node Gateway
   │
   ├── PostgreSQL
   ├── Redis
   └── Object Storage
          │
     ┌────┴────┐
     ▼         ▼
    C++      Python
   Worker    Worker
```

---

# 7. Unit Testing

Unit tests validate isolated behavior.

Every service must have unit tests.

---

# 8. Node.js Gateway Unit Tests

Recommended stack:

* Vitest or Jest
* TypeScript
* Supertest for HTTP-level testing

Test categories:

### Authentication

* password hashing
* password verification
* token generation
* token validation
* session expiration
* refresh-token rotation
* logout invalidation

### Authorization

Test:

```text
User A → Media A → allowed

User A → Media B → denied
```

Test:

* owner access
* unauthorized access
* missing authentication
* expired token
* invalid token
* role restrictions

### Media service

Test:

* media registration
* metadata validation
* MIME validation
* ownership
* soft deletion
* asset association

### Job service

Test:

* job creation
* priority
* status transitions
* idempotency
* retry calculation
* cancellation
* dependency validation

### Error handling

Verify that internal exceptions become stable API responses.

Example:

```json
{
  "error": {
    "code": "MEDIA_NOT_FOUND",
    "message": "Media resource was not found",
    "requestId": "..."
  }
}
```

---

# 9. Node.js API Test Cases

| Test                                 | Expected            |
| ------------------------------------ | ------------------- |
| Register valid user                  | 201                 |
| Register duplicate email             | 409                 |
| Login valid credentials              | 200                 |
| Login invalid credentials            | 401                 |
| Access protected route without token | 401                 |
| Access another user's media          | 403/404             |
| Create valid job                     | 202                 |
| Create duplicate idempotent job      | Same logical result |
| Cancel queued job                    | 200                 |
| Cancel completed job                 | Rejected            |
| Request nonexistent media            | 404                 |
| Invalid request body                 | 400/422             |

---

# 10. PostgreSQL Testing

Database testing must cover both schema correctness and application behavior.

## 10.1 Migration Tests

Every migration must be tested from:

```text
Empty Database
      ↓
Migration 001
      ↓
Migration 002
      ↓
...
      ↓
Current Schema
```

Verify:

* migrations apply successfully
* migrations are repeatable where designed
* constraints exist
* indexes exist
* foreign keys work
* triggers work
* rollback strategy is documented

---

# 11. Database Integrity Tests

Test:

### Users

* unique email
* valid timestamps
* session relationships

### Media

* valid owner
* soft deletion
* asset relationship

### Jobs

* valid media
* valid status
* valid priority
* attempt limits
* lease timestamps

### Dependencies

Prevent:

```text
Job A → Job A
```

and invalid dependency references.

### Results

Results must not reference nonexistent jobs.

---

# 12. Transaction Tests

Important transactions include:

```text
Create Media
      +
Create Asset
```

and:

```text
Create Job
      +
Persist Event
```

and:

```text
Complete Job
      +
Persist Result
      +
Publish Update
```

Tests should verify that partial failures do not leave inconsistent database state.

---

# 13. Redis Testing

Redis tests validate queue behavior rather than Redis itself.

Test:

* enqueue
* dequeue
* acknowledgment
* visibility/lease timeout
* retry
* duplicate delivery
* priority
* dead-letter behavior
* worker crash
* queue backlog
* concurrent consumers

Example:

```text
Producer
   │
   ▼
Redis
   │
 ┌─┴───────┐
 ▼         ▼
Worker A  Worker B
```

Both workers must not permanently process the same exclusive job.

---

# 14. Queue Concurrency Tests

A concurrency test should create:

```text
100 jobs
10 workers
```

and verify:

* all jobs eventually complete
* no job is lost
* successful jobs are not processed more than once unless explicitly allowed
* failed jobs retry according to policy
* attempt count remains correct

---

# 15. Worker Lease Tests

Worker leasing is critical to distributed reliability.

Test:

```text
Job queued
   ↓
Worker A leases job
   ↓
Worker A crashes
   ↓
Lease expires
   ↓
Worker B acquires job
   ↓
Job continues/retries
```

Verify that:

* expired jobs become recoverable
* active leases are not stolen prematurely
* heartbeat extends valid leases
* dead workers do not retain jobs indefinitely

---

# 16. Retry Tests

Test failure classes:

### Transient

```text
Storage timeout
Redis timeout
Temporary network failure
Temporary provider failure
```

Expected:

```text
Retry
```

### Permanent

```text
Unsupported media
Invalid manifest
Corrupted file
Invalid configuration
```

Expected:

```text
Fail
```

### Exhausted

```text
Attempt 1 → fail
Attempt 2 → fail
Attempt 3 → fail
Maximum attempts reached
       ↓
Dead-letter / terminal failure
```

---

# 17. Object Storage Tests

Test:

* object upload
* object download
* object existence
* generated object keys
* content type
* checksum
* private access
* signed URL generation
* expiration
* deletion
* missing object
* interrupted upload

Example key:

```text
media/{mediaId}/original/{assetId}
```

The server must generate the storage key rather than trusting user-supplied paths.

---

# 18. Upload Validation Tests

Test:

* valid MP4
* valid MOV
* valid WebM
* valid WAV
* valid MP3
* unsupported extension
* spoofed MIME type
* mismatched file signature
* empty file
* oversized file
* malformed container
* truncated file
* decompression/resource abuse
* dangerous filename
* path traversal attempt

Example malicious input:

```text
../../../../etc/passwd
```

must never become a filesystem path.

---

# 19. C++ Media Engine Testing

The C++ media engine is safety- and performance-critical.

Testing must include:

* unit tests
* integration tests
* FFmpeg integration
* malformed input tests
* memory-safety tests
* fuzzing
* benchmark tests

Recommended framework:

```text
GoogleTest
```

or another established C++ testing framework.

---

# 20. C++ Unit Test Categories

Test:

### Time utilities

```text
milliseconds
seconds
timestamps
frame numbers
```

### Media metadata

```text
duration
width
height
fps
codec
audio channels
sample rate
```

### Validation

* invalid duration
* negative timestamps
* invalid frame range
* unsupported codec

### Clip calculation

Input:

```text
start = 10.0
end = 25.0
```

Expected:

```text
duration = 15.0
```

Invalid:

```text
start >= end
```

must fail validation.

---

# 21. FFmpeg Integration Tests

Test native FFmpeg operations:

```text
Input
  ↓
Demux
  ↓
Decode
  ↓
Process
  ↓
Encode
  ↓
Mux
  ↓
Output
```

Test:

* MP4 probing
* codec detection
* stream discovery
* duration
* frame decoding
* audio decoding
* thumbnail extraction
* clipping
* audio extraction
* metadata extraction

---

# 22. Golden Media Corpus

A fixed media corpus must be maintained.

Recommended:

```text
tests/fixtures/media/
├── video-h264.mp4
├── video-h265.mp4
├── video-no-audio.mp4
├── video-audio.mp4
├── variable-fps.mp4
├── short-video.mp4
├── long-video.mp4
├── audio-mono.wav
├── audio-stereo.wav
├── image.jpg
├── malformed.mp4
└── truncated.mp4
```

Each fixture should have documented:

* codec
* resolution
* duration
* frame rate
* audio properties
* expected probe result
* checksum

---

# 23. Golden File Testing

For deterministic operations:

```text
Input
  ↓
C++ Processor
  ↓
Output
```

compare:

* metadata
* dimensions
* duration
* frame count
* audio properties
* checksums where deterministic

For encoded media, exact binary equality should not always be required because encoder metadata or container ordering may differ.

Prefer semantic comparison.

---

# 24. C++ Memory-Safety Testing

C++ CI should include:

```text
AddressSanitizer
UndefinedBehaviorSanitizer
ThreadSanitizer
```

where compatible.

Example build configuration:

```bash
cmake -S . -B build \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"
```

Then:

```bash
cmake --build build
ctest --test-dir build --output-on-failure
```

Tests must fail on:

* buffer overflow
* use-after-free
* double free
* invalid memory access
* undefined behavior
* data races where detected

---

# 25. C++ Fuzz Testing

Fuzz targets should focus on:

* media metadata parsing
* manifest parsing
* timestamp validation
* codec/container handling
* command/input parsing
* malformed media handling

Basic invariant:

> Arbitrary input must never crash the worker process or violate memory safety.

Fuzzing should run:

* locally during development
* scheduled in CI
* continuously for critical parsers where practical

---

# 26. Python Worker Testing

Recommended tools:

* pytest
* pytest-asyncio where required
* Ruff
* mypy where adopted
* hypothesis for property-based tests

Test:

* artifact loading
* manifest parsing
* preprocessing
* transcription
* segmentation
* sentiment analysis
* detection adapters
* AI provider adapters
* result normalization
* retries
* error classification

---

# 27. AI/ML Testing

AI outputs require different validation from deterministic software.

Tests should verify:

### Schema correctness

Output must conform to:

```json
{
  "model": "model-name",
  "modelVersion": "1.0",
  "confidence": 0.92,
  "results": []
}
```

### Range validation

Confidence:

```text
0.0 <= confidence <= 1.0
```

### Deterministic preprocessing

The same input should produce the same preprocessing output.

### Provider failure

Simulate:

```text
timeout
rate limit
invalid response
provider unavailable
```

and verify correct classification.

---

# 28. AI Model Regression Tests

Maintain a small fixed evaluation dataset.

Example:

```text
tests/fixtures/ai/
├── transcription/
├── sentiment/
├── detection/
└── classification/
```

Track:

* precision
* recall
* F1
* transcription error rate where applicable
* confidence distribution
* latency

Model changes must be evaluated against the baseline.

---

# 29. Artifact and Manifest Contract Testing

The C++ and Python services communicate through artifacts and manifests.

This boundary must be strongly tested.

Example:

```text
C++ Worker
    │
    ▼
manifest.json
    │
    ▼
Python Worker
```

The manifest must contain:

* schema version
* job ID
* media ID
* input asset
* artifact references
* metadata
* checksums
* timestamps
* producer version

Test:

* valid manifest
* missing fields
* wrong types
* unknown version
* unsupported version
* invalid artifact reference
* invalid checksum
* duplicate artifact
* corrupted JSON

---

# 30. Manifest Compatibility

Version:

```text
manifestVersion = 1
```

When version 2 is introduced:

```text
Version 1 → supported
Version 2 → supported
Version 3 → rejected gracefully
```

Workers must never silently interpret an unsupported schema.

---

# 31. API Contract Testing

The API contract should be represented using OpenAPI.

Contract tests validate:

* request schema
* response schema
* status codes
* required fields
* pagination
* authentication requirements
* error schema

Example:

```text
OpenAPI
   │
   ├── Node implementation
   │
   └── Automated contract tests
```

The API implementation must remain compatible with the documented contract.

---

# 32. WebSocket Testing

Test:

* connection authentication
* subscription
* job progress
* completion
* failure
* cancellation
* reconnect
* duplicate event handling
* event ordering where required

Example:

```text
Client
  │
  │ connect
  ▼
WebSocket
  │
  ├── job.started
  ├── job.progress
  ├── job.progress
  └── job.completed
```

The frontend must not break if a connection temporarily disappears.

---

# 33. Event Contract

Every event should contain:

```json
{
  "event": "job.progress",
  "jobId": "uuid",
  "timestamp": "2026-01-01T12:00:00Z",
  "correlationId": "uuid",
  "payload": {}
}
```

Contract tests validate:

* event name
* schema
* required fields
* timestamp format
* identifier validity

---

# 34. React Unit Testing

Recommended:

* Vitest
* React Testing Library

Test:

### Components

* upload form
* media list
* media player
* timeline
* job status
* progress bar
* transcript viewer
* detection overlay
* clip editor
* dashboard cards

### Hooks

* authentication
* media fetching
* job polling
* WebSocket subscription
* upload progress

---

# 35. Frontend Integration Testing

Test complete UI flows:

```text
Login
 ↓
Dashboard
 ↓
Upload Media
 ↓
Create Processing Job
 ↓
Observe Progress
 ↓
Open Result
 ↓
View Transcript
 ↓
View Detections
```

Verify loading, success, empty, error, and retry states.

---

# 36. Frontend Error-State Testing

Every asynchronous operation must have tests for:

```text
Loading
Success
Empty
Error
Retry
Unauthorized
Network Failure
Timeout
```

Example:

```text
Upload
 ├── uploading
 ├── completed
 ├── failed
 └── cancelled
```

---

# 37. Accessibility Testing

The frontend should test:

* keyboard navigation
* focus behavior
* semantic controls
* labels
* accessible names
* contrast
* dialog behavior
* screen-reader-friendly status updates

Automated accessibility testing should be included where practical.

---

# 38. Integration Testing

Integration tests verify real boundaries.

Primary combinations:

```text
Node ↔ PostgreSQL
Node ↔ Redis
Node ↔ Object Storage
Node ↔ C++ Worker
Node ↔ Python Worker
C++ ↔ Object Storage
Python ↔ Object Storage
React ↔ Node
Node ↔ WebSocket
```

---

# 39. Disposable Infrastructure

Integration tests should preferably use disposable services.

Example:

```text
Test Runner
    │
    ├── PostgreSQL container
    ├── Redis container
    └── Object Storage container
```

At the end:

```text
Test Complete
     ↓
Containers Destroyed
     ↓
Clean Environment
```

Tests must not depend on developer-specific persistent state.

---

# 40. End-to-End Testing

The most important E2E test is the golden path.

```text
Register
   ↓
Login
   ↓
Upload Media
   ↓
Create Job
   ↓
Queue
   ↓
C++ Processing
   ↓
Artifact Manifest
   ↓
Python Processing
   ↓
Persist Results
   ↓
WebSocket Update
   ↓
React Dashboard
```

Expected result:

```text
Job = completed
Artifacts = available
Results = persisted
UI = updated
```

---

# 41. E2E Golden Test

Example scenario:

### Input

```text
short-video.mp4
```

### Expected

```text
Media registered
Original uploaded
Job created
C++ worker completes
Metadata generated
Thumbnail generated
Manifest generated
Python worker processes manifest
Results persisted
WebSocket completion event emitted
Frontend displays result
```

This test is the minimum proof that the entire architecture works.

---

# 42. Failure Injection Testing

Distributed systems must test failures intentionally.

Inject:

### Database failure

```text
PostgreSQL unavailable
```

Expected:

* API returns controlled error
* no process crash
* readiness reports unhealthy

### Redis failure

Expected:

* job creation fails safely or enters defined recovery path
* no false job completion

### Object storage failure

Expected:

* upload failure reported
* incomplete job does not become completed

### Worker failure

Expected:

```text
Worker A crashes
      ↓
Lease expires
      ↓
Worker B recovers job
```

---

# 43. Chaos-Lite Tests

The project does not initially require a full chaos-engineering platform.

Instead implement controlled fault injection:

```text
Kill worker
Delay Redis
Reject storage request
Timeout AI provider
Restart PostgreSQL
Disconnect WebSocket
```

Verify system recovery.

---

# 44. Concurrency Testing

Concurrency tests are mandatory for:

* job leasing
* job cancellation
* retries
* duplicate messages
* artifact creation
* database updates
* WebSocket events

Example:

```text
10 workers
100 jobs
```

Expected:

```text
100 terminal job states
0 lost jobs
0 impossible state transitions
```

---

# 45. Race Condition Tests

Important races:

### Cancel vs Start

```text
Cancel Job
     ↕
Worker Starts Job
```

System must resolve the race according to the documented state machine.

### Retry vs Completion

```text
Retry Request
     ↕
Worker Completion
```

Only one terminal result should win.

### Duplicate completion

```text
Worker A → complete
Worker A → complete
```

Must remain idempotent.

---

# 46. Idempotency Testing

Operations requiring idempotency:

* media registration
* job creation
* job completion
* result persistence
* artifact registration

Example:

```text
POST /jobs
Idempotency-Key: abc123
```

Repeated request:

```text
same logical job
```

rather than:

```text
Job 1
Job 2
Job 3
```

---

# 47. Security Testing

Security tests follow the security architecture defined in `07-security.md`.

Test:

### Authentication

* invalid token
* expired token
* revoked session
* malformed token

### Authorization

* cross-user media access
* cross-user job access
* administrative endpoint access

### Input validation

* SQL injection
* path traversal
* oversized values
* malformed JSON
* invalid UUID
* invalid timestamps

---

# 48. Upload Security Tests

Test:

```text
fake.mp4
```

containing non-video content.

The system must validate actual content rather than relying exclusively on:

```text
filename
extension
Content-Type
```

Test:

```text
malicious_filename.mp4
../../../file.mp4
```

and verify generated object/file paths remain safe.

---

# 49. API Security Tests

Automate checks for:

* authentication bypass
* IDOR
* privilege escalation
* excessive request size
* rate-limit bypass
* malformed JSON
* parameter pollution
* CORS misconfiguration
* SSRF-sensitive endpoints
* verbose error leakage

---

# 50. Dependency Security

CI should scan:

### Node

```text
npm audit
```

### Python

Dependency vulnerability scanner.

### C++

Dependency/package scanning where available.

### Containers

Image vulnerability scanning.

### Repository

Secret scanning.

No production secret may exist in:

```text
source
.git
Dockerfile
logs
test fixtures
CI artifacts
```

---

# 51. Performance Testing

Performance testing is especially important for the C++ media engine.

Measure:

* media probe latency
* decode throughput
* clip generation throughput
* thumbnail extraction
* audio extraction
* memory usage
* CPU utilization
* queue latency
* end-to-end processing latency

---

# 52. Performance Metrics

Important metrics:

```text
API latency
P50
P95
P99

Queue wait time

Job processing time

Artifact generation time

Python inference latency

End-to-end latency

CPU utilization

Memory usage

Throughput
```

---

# 53. C++ Benchmarking

Example:

```text
Input:
1920x1080 H.264
Duration:
60 seconds
```

Measure:

```text
probe time
decode time
thumbnail time
clip time
audio extraction time
peak RSS
CPU time
```

Benchmarks should be repeatable.

---

# 54. API Load Testing

Example baseline:

```text
100 concurrent clients
```

Operations:

```text
GET /media
GET /jobs
GET /dashboard
POST /jobs
```

Measure:

* throughput
* latency
* error rate
* database CPU
* connection pool usage

---

# 55. Queue Load Testing

Example:

```text
10,000 jobs
50 workers
```

Measure:

* enqueue throughput
* dequeue throughput
* average queue delay
* maximum queue delay
* completion rate
* retry rate
* worker utilization

---

# 56. Stress Testing

Stress tests push beyond normal expected load.

Example:

```text
Normal:
100 jobs/minute

Stress:
1,000 jobs/minute
```

Observe:

* queue growth
* database behavior
* memory growth
* worker saturation
* recovery after load decreases

---

# 57. Soak Testing

Run the system for an extended period:

```text
4–24 hours
```

with continuous jobs.

Detect:

* memory leaks
* resource leaks
* stale workers
* queue buildup
* connection leaks
* database growth problems
* degraded latency

---

# 58. Resource Limit Testing

Workers must be tested against:

```text
CPU limit
Memory limit
Disk limit
Execution timeout
File-size limit
Artifact count limit
```

Example:

```text
Large Media
    ↓
Resource Limit
    ↓
Controlled Failure
```

The system must not allow one job to exhaust the entire host.

---

# 59. Test Data Strategy

Test data must be:

* deterministic
* versioned
* small enough for CI
* representative
* legally distributable
* free of sensitive personal data

Avoid large media files in Git.

Large datasets should use:

```text
artifact storage
CI cache
release assets
dedicated test dataset storage
```

---

# 60. Test Fixture Metadata

Every media fixture should have metadata:

```json
{
  "file": "video-h264.mp4",
  "sha256": "...",
  "durationSeconds": 12.4,
  "width": 1920,
  "height": 1080,
  "fps": 30,
  "videoCodec": "h264",
  "audioCodec": "aac"
}
```

This allows automated validation.

---

# 61. Deterministic Testing

Tests should avoid:

* current time where unnecessary
* random UUIDs without control
* random media generation without seed
* external AI services
* production databases
* public APIs

Use:

```text
fake clock
seeded randomness
mock providers
isolated databases
local object storage
```

---

# 62. External AI Provider Testing

Never make normal CI depend on an external AI provider.

Instead:

```text
Python Worker
     │
     ▼
Provider Interface
     │
 ┌───┴──────────┐
 ▼              ▼
Mock Provider   Real Provider
```

CI uses the mock provider.

Scheduled integration tests may use a real provider if credentials, cost, privacy, and rate limits are controlled.

---

# 63. Coverage Targets

Recommended minimum targets:

| Component              | Line Coverage | Branch Coverage |
| ---------------------- | ------------: | --------------: |
| Node Gateway           |         ≥ 80% |           ≥ 75% |
| C++ Core               |         ≥ 85% |           ≥ 80% |
| Python Worker          |         ≥ 80% |           ≥ 75% |
| React                  |         ≥ 75% |           ≥ 70% |
| Critical Security Code |         ≥ 90% |           ≥ 85% |

Coverage is a quality signal, not the definition of correctness.

A test suite with 95% coverage can still miss important distributed-system failures.

---

# 64. Critical-Path Coverage

The following behaviors must have automated tests regardless of percentage:

```text
Authentication
Authorization
Media ownership
Upload validation
Job creation
Job leasing
Retry
Cancellation
Worker recovery
Artifact validation
Result persistence
Idempotency
WebSocket completion
```

---

# 65. State-Machine Testing

The job state machine must be tested independently.

Example:

```text
queued
  ↓
leased
  ↓
running
  ↓
completed
```

Alternative:

```text
queued
  ↓
leased
  ↓
running
  ↓
failed
  ↓
retrying
  ↓
queued
```

Invalid transitions must be rejected.

Example:

```text
completed → running
```

must fail.

---

# 66. Property-Based Testing

Property-based tests are useful for:

* timestamps
* clip intervals
* pagination
* job retry calculations
* bounding boxes
* confidence values
* manifest validation

Example property:

```text
For every valid clip:
start < end
```

Another:

```text
For every bounding box:
0 <= x <= 1
0 <= y <= 1
0 <= width <= 1
0 <= height <= 1
```

---

# 67. API Pagination Testing

Test:

```text
page size = 1
page size = 10
page size = maximum
page size = invalid
```

Verify:

* stable ordering
* no duplicate records
* no missing records
* correct next cursor/page information

---

# 68. Time-Based Testing

Test:

* expired sessions
* job lease expiration
* retry delays
* signed URL expiration
* stale worker detection
* timestamp ordering

Use a controllable test clock where possible.

---

# 69. Health Check Testing

### Liveness

```text
GET /health/live
```

should indicate whether the process is alive.

### Readiness

```text
GET /health/ready
```

must reflect required dependencies.

Test scenarios:

```text
PostgreSQL available
PostgreSQL unavailable

Redis available
Redis unavailable

Object storage available
Object storage unavailable
```

---

# 70. Graceful Shutdown Testing

Test:

```text
Worker receives SIGTERM
        ↓
Stops accepting jobs
        ↓
Finishes safe work or releases lease
        ↓
Flushes logs/events
        ↓
Closes connections
        ↓
Exits
```

Verify no job becomes permanently lost.

---

# 71. Worker Registration Testing

Test:

* worker registration
* capability declaration
* heartbeat
* stale worker detection
* worker shutdown
* capability mismatch

Example:

```json
{
  "workerType": "cpp-media",
  "capabilities": [
    "probe",
    "thumbnail",
    "clip",
    "audio_extract"
  ]
}
```

A worker must not receive unsupported jobs.

---

# 72. Contract Compatibility Matrix

Maintain compatibility between:

```text
Node Gateway
C++ Worker
Python Worker
Manifest
Database
Frontend
```

Example:

| Contract        | Producer | Consumer |
| --------------- | -------- | -------- |
| REST API        | Node     | React    |
| Job Message     | Node     | Workers  |
| Manifest        | C++      | Python   |
| Result Schema   | Python   | Node     |
| WebSocket Event | Node     | React    |

Every contract change must update its tests.

---

# 73. Regression Testing

Every production bug should result in:

```text
Bug
 ↓
Root Cause
 ↓
Regression Test
 ↓
Fix
 ↓
CI
```

A bug must not be considered permanently fixed until a test reproduces the original failure.

---

# 74. Flaky Test Policy

Flaky tests are defects.

A test is considered flaky when:

```text
same code
same input
different result
```

Actions:

1. identify root cause
2. quarantine only when necessary
3. create tracking issue
4. fix quickly
5. restore test to mandatory CI

Tests must not be permanently disabled to make CI green.

---

# 75. Test Isolation

Tests must not depend on execution order.

Bad:

```text
test_A creates user
test_B assumes user exists
```

Good:

```text
test_B creates its own user
```

Each test should establish its own required state.

---

# 76. Test Cleanup

Integration tests must clean:

* database records
* Redis queues
* object-storage objects
* temporary files
* worker state

Preferred approach:

```text
Disposable environment
```

rather than complex cleanup against a shared environment.

---

# 77. CI Pipeline

Recommended pipeline:

```text
                    ┌──────────────┐
                    │   Checkout   │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ Dependencies │
                    └──────┬───────┘
                           ▼
                 ┌────────────────────┐
                 │ Format + Lint      │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ Unit Tests         │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ Contract Tests     │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ Integration Tests  │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ Security Tests     │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ Build              │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ E2E Tests          │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ Release Gate       │
                 └────────────────────┘
```

---

# 78. CI Test Stages

## Stage 1 — Fast Checks

Run on every push:

```bash
npm run lint
npm test
pytest
ctest
```

plus formatting and type checks.

---

## Stage 2 — Unit

Run:

```text
Node unit tests
C++ unit tests
Python unit tests
React unit tests
```

---

## Stage 3 — Integration

Start:

```bash
docker compose up -d postgres redis object-storage
```

Run integration tests.

---

## Stage 4 — E2E

Start the full stack:

```bash
docker compose up -d
```

Then run the golden workflow.

---

# 79. Example Test Commands

## Node

```bash
cd gateway-node

npm test
npm run test:coverage
npm run lint
```

## C++

```bash
cd media-engine-cpp

cmake -S . -B build
cmake --build build -j
ctest --test-dir build --output-on-failure
```

## Python

```bash
cd analytics-python

source .venv/bin/activate

pytest
pytest --cov
ruff check .
```

## React

```bash
cd frontend-react

npm test
npm run lint
npm run build
```

---

# 80. Root Test Command

The root project should eventually provide:

```bash
make test
```

which executes:

```text
Node tests
+
C++ tests
+
Python tests
+
React tests
+
integration tests
```

Additional commands:

```bash
make test-unit
make test-integration
make test-e2e
make test-security
make test-performance
make coverage
```

---

# 81. Test Reports

CI should retain:

* JUnit reports
* coverage reports
* sanitizer logs
* fuzzing results
* E2E screenshots
* E2E videos where configured
* performance reports
* security scan reports
* failed test logs

Reports should be associated with the commit or pull request.

---

# 82. Failure Diagnostics

A failed test should provide:

```text
Test Name
Environment
Commit SHA
Request ID
Correlation ID
Input Fixture
Expected Result
Actual Result
Relevant Logs
Stack Trace
Artifact References
```

Distributed failures should be traceable using the same correlation ID across services.

---

# 83. Observability During Tests

Integration and E2E tests should preserve:

```text
requestId
correlationId
jobId
mediaId
workerId
attemptId
```

Example:

```text
API Request
   │ correlationId=abc
   ▼
Job
   │ correlationId=abc
   ▼
C++ Worker
   │ correlationId=abc
   ▼
Manifest
   │ correlationId=abc
   ▼
Python Worker
```

This makes distributed test failures diagnosable.

---

# 84. Test Logging Rules

Tests should log enough information to debug failures but must never log:

* passwords
* access tokens
* refresh tokens
* secret keys
* private signed URLs
* sensitive media content

Use redacted identifiers where appropriate.

---

# 85. Release Quality Gates

A release candidate must satisfy:

### Functional

* all critical E2E tests pass
* no P0/P1 functional defects

### Security

* no known critical vulnerabilities
* authentication tests pass
* authorization tests pass
* upload security tests pass
* secret scanning passes

### Reliability

* retry tests pass
* worker recovery passes
* idempotency tests pass
* graceful shutdown passes

### Performance

* benchmark thresholds pass
* no severe regression from baseline

### Quality

* required coverage thresholds pass
* static analysis passes
* no unexplained flaky tests

---

# 86. Quality Gate Matrix

| Gate           | Development | PR       | Release  |
| -------------- | ----------- | -------- | -------- |
| Formatting     | Required    | Required | Required |
| Lint           | Required    | Required | Required |
| Unit Tests     | Required    | Required | Required |
| Contract Tests | Recommended | Required | Required |
| Integration    | Optional    | Required | Required |
| E2E            | Optional    | Required | Required |
| Security       | Basic       | Required | Full     |
| Fuzzing        | Scheduled   | Selected | Full     |
| Performance    | Local       | Smoke    | Full     |
| Soak           | No          | No       | Required |

---

# 87. Defect Severity

## P0 — Critical

Examples:

* authentication bypass
* data corruption
* arbitrary code execution
* complete pipeline failure

Release blocker.

## P1 — High

Examples:

* job loss
* cross-user data access
* persistent worker failure
* major media-processing failure

Release blocker.

## P2 — Medium

Examples:

* non-critical API failure
* degraded UI
* recoverable processing issue

Normally fix before release.

## P3 — Low

Examples:

* minor UI defect
* documentation issue
* cosmetic behavior

Can enter backlog.

---

# 88. Test Prioritization

When development time is limited, prioritize:

```text
1. Security
2. Data integrity
3. Job lifecycle
4. Worker recovery
5. Media correctness
6. API contracts
7. E2E golden path
8. Performance
9. UI edge cases
10. Cosmetic behavior
```

---

# 89. Testing by Development Phase

## Phase 0 — Repository

Test:

* build
* lint
* formatting
* CI

## Phase 1 — Database/Gateway

Test:

* schema
* database layer
* health
* API

## Phase 2 — Authentication

Test:

* registration
* login
* refresh
* logout
* authorization

## Phase 3 — Storage

Test:

* upload
* validation
* signed URLs
* checksums

## Phase 4 — Queue

Test:

* enqueue
* lease
* retry
* cancellation
* recovery

## Phase 5 — C++

Test:

* FFmpeg probe
* metadata
* thumbnail
* clipping
* audio

## Phase 6 — Python

Test:

* manifest
* preprocessing
* transcription
* analytics
* result schema

## Phase 7 — E2E

Test:

```text
Upload → Process → Analyze → Display
```

## Phase 8 — Frontend

Test:

* media studio
* timeline
* upload
* job progress
* results

## Phase 9 — Realtime

Test:

* WebSocket
* reconnect
* event ordering

## Phase 10+ — Production

Test:

* security
* performance
* recovery
* deployment
* backup/restore

---

# 90. Acceptance Test Matrix

| Capability         | Unit | Integration | E2E |
| ------------------ | ---: | ----------: | --: |
| Authentication     |    ✓ |           ✓ |   ✓ |
| Authorization      |    ✓ |           ✓ |   ✓ |
| Media Registration |    ✓ |           ✓ |   ✓ |
| Upload             |    ✓ |           ✓ |   ✓ |
| Object Storage     |    ✓ |           ✓ |   ✓ |
| Job Creation       |    ✓ |           ✓ |   ✓ |
| Queue              |    ✓ |           ✓ |   ✓ |
| Worker Lease       |    ✓ |           ✓ |   ✓ |
| C++ Probe          |    ✓ |           ✓ |   ✓ |
| Thumbnail          |    ✓ |           ✓ |   ✓ |
| Clip               |    ✓ |           ✓ |   ✓ |
| Audio Extraction   |    ✓ |           ✓ |   ✓ |
| Manifest           |    ✓ |           ✓ |   ✓ |
| Python Analytics   |    ✓ |           ✓ |   ✓ |
| Result Persistence |    ✓ |           ✓ |   ✓ |
| WebSocket          |    ✓ |           ✓ |   ✓ |
| Dashboard          |    ✓ |           ✓ |   ✓ |

---

# 91. Definition of Done for Tests

A feature is not complete until:

* unit tests exist
* integration tests exist where boundaries are involved
* contract tests are updated
* failure paths are tested
* security implications are tested
* relevant coverage thresholds pass
* E2E test is updated when the golden path changes
* documentation is updated
* CI passes

---

# 92. Testing Checklist for New Features

```text
[ ] Requirements identified
[ ] Unit tests written
[ ] Happy path tested
[ ] Error path tested
[ ] Boundary conditions tested
[ ] Authorization tested
[ ] Idempotency considered
[ ] Concurrency considered
[ ] Integration tests added
[ ] Contract updated
[ ] E2E updated if required
[ ] Security reviewed
[ ] Performance impact measured
[ ] Logs are testable
[ ] Documentation updated
```

---

# 93. Example: New Media Clip Feature

Suppose a new endpoint is introduced:

```text
POST /api/v1/media/:mediaId/clips
```

Testing must include:

### Unit

```text
validate start
validate end
validate media ownership
validate clip parameters
```

### API

```text
valid request → 201/202
invalid request → 400/422
unauthorized → 401
forbidden → 403/404
missing media → 404
```

### C++

```text
decode
seek
clip
encode
mux
```

### Integration

```text
API
 ↓
Postgres
 ↓
Redis
 ↓
C++
 ↓
Object Storage
```

### E2E

```text
Open Media
 ↓
Create Clip
 ↓
Render
 ↓
Observe Progress
 ↓
Open Result
```

---

# 94. Example: Worker Crash Test

Scenario:

```text
Job J1
  ↓
Worker A
  ↓
lease acquired
  ↓
Worker A terminated
```

Expected:

```text
lease expires
      ↓
J1 becomes recoverable
      ↓
Worker B acquires J1
      ↓
processing continues
      ↓
J1 completed
```

Assertions:

```text
job.status = completed
attempt_count >= 1
no permanent orphan lease
result exists
completion event exists
```

---

# 95. Example: Duplicate Completion Test

Worker sends:

```text
complete(J1)
complete(J1)
```

Expected:

```text
one logical result
one terminal job state
no duplicate artifacts
```

This validates idempotency.

---

# 96. Example: Malicious Upload Test

Input:

```text
filename = ../../../../tmp/evil.mp4
Content-Type = video/mp4
actual content = invalid
```

Expected:

```text
Upload rejected
No arbitrary filesystem write
No unsafe object key
No worker execution
Security event logged
```

---

# 97. Example: Database Failure Test

During job completion:

```text
Worker
  ↓
Complete job
  ↓
PostgreSQL unavailable
```

Expected:

```text
Completion is not falsely acknowledged
Worker retries according to policy
Job remains recoverable
No corrupted terminal state
```

---

# 98. Example: WebSocket Reconnection Test

Scenario:

```text
Job running
   ↓
WebSocket disconnected
   ↓
Job progresses
   ↓
Client reconnects
```

Expected:

```text
Client receives current state
```

The UI must not remain permanently stuck on an old progress value.

---

# 99. Performance Regression Policy

Maintain baseline benchmarks.

Example:

```text
Baseline C++ probe:
120 ms

New implementation:
125 ms
```

Small variance may be acceptable.

But:

```text
120 ms → 300 ms
```

must trigger investigation.

Performance thresholds should be versioned with the benchmark suite.

---

# 100. Testing Documentation

The project should maintain:

```text
docs/
├── 09-testing-strategy.md
└── testing/
    ├── test-data.md
    ├── benchmark-baseline.md
    ├── e2e-scenarios.md
    ├── failure-injection.md
    └── ci-testing.md
```

These documents can be introduced as the implementation matures.

---

# 101. Recommended Testing Toolchain

| Area                    | Recommended Tool                 |
| ----------------------- | -------------------------------- |
| Node unit               | Vitest                           |
| Node HTTP               | Supertest                        |
| C++ unit                | GoogleTest                       |
| C++ memory              | ASan/UBSan                       |
| C++ race                | TSan                             |
| C++ fuzzing             | libFuzzer                        |
| Python                  | pytest                           |
| Python property testing | Hypothesis                       |
| Python lint             | Ruff                             |
| React                   | Vitest                           |
| React components        | React Testing Library            |
| API contract            | OpenAPI-based validation         |
| Browser E2E             | Playwright                       |
| PostgreSQL              | Testcontainers/disposable DB     |
| Redis                   | Testcontainers/disposable Redis  |
| Object Storage          | Local S3-compatible service      |
| Load testing            | k6                               |
| Container testing       | Docker Compose                   |
| Security                | SAST/SCA/container scanning      |
| Coverage                | language-specific coverage tools |

Tool selection may evolve as implementation begins.

---

# 102. Minimum MVP Test Suite

Before calling the MVP functional, the following must pass:

```text
Authentication
    ✓ Register
    ✓ Login
    ✓ Authorization

Media
    ✓ Register
    ✓ Upload
    ✓ Retrieve

Jobs
    ✓ Create
    ✓ Queue
    ✓ Lease
    ✓ Complete
    ✓ Retry
    ✓ Cancel

C++
    ✓ Probe
    ✓ Metadata
    ✓ Thumbnail

Python
    ✓ Manifest
    ✓ Basic analysis
    ✓ Result schema

Integration
    ✓ PostgreSQL
    ✓ Redis
    ✓ Object Storage

Frontend
    ✓ Login
    ✓ Upload
    ✓ Job progress
    ✓ Result display

E2E
    ✓ Upload → Process → Display

Security
    ✓ Auth
    ✓ Authorization
    ✓ Upload validation

Reliability
    ✓ Worker recovery
    ✓ Idempotency
```

---

# 103. Production Test Suite

Before production release:

```text
Unit
Contract
Integration
E2E
Security
Fuzz
Memory Safety
Concurrency
Performance
Load
Stress
Soak
Failure Injection
Backup/Restore
Deployment
Rollback
Graceful Shutdown
```

All P0/P1 failures must be resolved.

---

# 104. Final Testing Architecture

```text
                    ┌─────────────────────────┐
                    │       Test Runner       │
                    └────────────┬────────────┘
                                 │
             ┌───────────────────┼───────────────────┐
             ▼                   ▼                   ▼
        Unit Tests          Contract Tests      Security Tests
             │                   │                   │
             └───────────────────┼───────────────────┘
                                 ▼
                       Integration Tests
                                 │
             ┌───────────────────┼───────────────────┐
             ▼                   ▼                   ▼
        PostgreSQL             Redis             Storage
             │                   │                   │
             └───────────────────┼───────────────────┘
                                 ▼
                          Worker Tests
                           ┌─────┴─────┐
                           ▼           ▼
                          C++        Python
                           │           │
                           └─────┬─────┘
                                 ▼
                           E2E Pipeline
                                 │
                                 ▼
                         React / WebSocket
                                 │
                                 ▼
                         Acceptance Tests
```

---

# 105. Testing Roadmap

## Sprint 1

```text
Test framework setup
CI test pipeline
Node unit tests
C++ unit tests
Python unit tests
React unit tests
```

## Sprint 2

```text
Database integration tests
API integration tests
Redis tests
Object-storage tests
```

## Sprint 3

```text
Worker lease tests
Retry tests
C++ FFmpeg tests
Manifest contract tests
```

## Sprint 4

```text
Python analytics tests
E2E golden path
WebSocket tests
```

## Sprint 5

```text
Security testing
Fuzzing
Sanitizers
Concurrency testing
```

## Sprint 6

```text
Performance benchmarks
Load tests
Stress tests
Soak tests
```

## Sprint 7

```text
Production-readiness validation
Failure injection
Backup/restore
Deployment tests
Release candidate validation
```

---

# 106. Final Acceptance Criteria

The testing strategy is considered implemented when:

```text
[✓] Unit test frameworks configured
[✓] CI executes automated tests
[✓] Database integration tests exist
[✓] Redis integration tests exist
[✓] Object-storage tests exist
[✓] API contract tests exist
[✓] Worker contract tests exist
[✓] C++ FFmpeg tests exist
[✓] Python worker tests exist
[✓] Manifest compatibility tests exist
[✓] React tests exist
[✓] WebSocket tests exist
[✓] E2E golden path exists
[✓] Failure injection exists
[✓] Security tests exist
[✓] C++ sanitizers are configured
[✓] Fuzzing strategy exists
[✓] Concurrency tests exist
[✓] Performance benchmarks exist
[✓] Coverage thresholds exist
[✓] Release quality gates exist
[✓] Flaky-test policy exists
[✓] Test reports are retained
```

---

# 107. Final Testing Principle

The platform should not be considered reliable merely because:

```text
the application starts
```

or because:

```text
the happy path works
```

A distributed media-processing system is reliable only when it can demonstrate:

```text
Correctness
     +
Security
     +
Concurrency Safety
     +
Failure Recovery
     +
Performance
     +
Observability
     +
Reproducibility
```

The final standard is therefore:

> **If a worker crashes, a queue fails, a database becomes unavailable, media is malformed, a request is duplicated, or a client disconnects, the system must fail predictably, recover where designed, and never silently corrupt state.**

That principle governs the entire testing strategy of the High-Performance Distributed Media Analytics Platform.
