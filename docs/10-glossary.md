# High-Performance Distributed Media Analytics Platform

# Glossary

**Document:** `10-glossary.md`
**Version:** 1.0
**Status:** Documentation Baseline
**Author:** Adarsh Kumar
**Project:** High-Performance Distributed Media Analytics Platform

---

## 1. Purpose

This glossary defines the technical terminology used throughout the High-Performance Distributed Media Analytics Platform.

The platform combines:

* C++
* FFmpeg
* Python
* AI/ML
* Node.js
* TypeScript
* React
* PostgreSQL
* Redis
* Object Storage
* WebSockets
* Docker
* Distributed workers
* Asynchronous job processing

Because these technologies use terminology from different engineering domains, this document establishes a consistent vocabulary for the project.

---

# 2. General Architecture Terms

## API

**Application Programming Interface.**

A defined interface through which software components communicate.

In this project, the primary API is the REST API exposed by the Node.js gateway.

---

## API Gateway

The Node.js + TypeScript service responsible for exposing the public HTTP API.

Responsibilities include:

* authentication
* authorization
* request validation
* media management
* job creation
* database access
* queue interaction
* result retrieval
* WebSocket coordination

---

## Artifact

A generated file or machine-readable output produced during media processing.

Examples:

* thumbnail
* extracted audio
* generated clip
* transcript file
* intermediate media
* analysis output

Artifacts are stored in object storage rather than PostgreSQL.

---

## Artifact Reference

A logical reference to an artifact stored outside the database.

Example:

```text
media/{mediaId}/artifacts/{artifactId}
```

The database stores metadata about the artifact while the actual binary remains in object storage.

---

## Control Plane

The part of the platform responsible for coordination and management.

```text
React
  ↓
Node.js
  ↓
PostgreSQL
Redis
Object Storage
```

The control plane manages:

* users
* media metadata
* jobs
* workers
* permissions
* state
* APIs

---

## Data Plane

The part of the system responsible for computational media processing.

```text
C++ Workers
Python Workers
FFmpeg
AI/ML
```

The data plane performs expensive processing rather than coordinating the application.

---

## Distributed System

A system composed of multiple independently executing components that communicate over defined interfaces.

This project is distributed because:

* API and workers are separate processes
* workers may run on different machines
* Redis coordinates asynchronous work
* PostgreSQL stores shared state
* object storage stores media artifacts

---

## Service Boundary

A defined boundary between independently deployable or logically isolated components.

Example:

```text
Node Gateway
      │
      │ Queue
      ▼
C++ Worker
```

---

## Polyglot Architecture

An architecture that intentionally uses multiple programming languages.

This project uses:

```text
TypeScript → API + Frontend
C++       → Media Engine
Python    → Analytics + AI/ML
SQL       → Database
```

Each language is selected for a specific workload.

---

## Vertical Slice

A small feature implemented across the entire technology stack.

Example:

```text
React
 ↓
Node
 ↓
PostgreSQL
 ↓
Redis
 ↓
C++
 ↓
Object Storage
 ↓
Node
 ↓
React
```

A vertical slice proves that the complete system works rather than only one isolated component.

---

# 3. Media Processing Terms

## Media

A digital audio, video, or image resource uploaded to the platform.

Examples:

* MP4
* MOV
* WebM
* WAV
* MP3
* JPEG

---

## Media Asset

A physical representation of a media object.

A single logical media record may have multiple assets.

Example:

```text
Media
├── Original
├── Thumbnail
├── Extracted Audio
└── Rendered Clip
```

---

## Media Container

A file format that packages one or more media streams.

Examples:

```text
MP4
MKV
MOV
WebM
AVI
```

A container is not the same thing as a codec.

---

## Codec

A technology used to encode and/or decode media.

Examples:

### Video

```text
H.264
H.265 / HEVC
VP9
AV1
```

### Audio

```text
AAC
MP3
Opus
PCM
```

---

## Encoding

Converting raw or decoded media into a compressed representation.

```text
Raw Frames
    ↓
Encoder
    ↓
H.264
```

---

## Decoding

Converting encoded media into a form that can be processed.

```text
H.264
  ↓
Decoder
  ↓
Raw Frames
```

---

## Demuxing

Extracting individual media streams from a container.

Example:

```text
video.mp4
   │
   ├── Video Stream
   └── Audio Stream
```

---

## Muxing

Combining encoded streams into a container.

```text
Video Stream
      +
Audio Stream
      ↓
     MP4
```

---

## Transcoding

Decoding media and encoding it again into another format, codec, bitrate, resolution, or configuration.

---

## Remuxing

Changing the container without necessarily re-encoding the underlying streams.

Example:

```text
H.264 + AAC
     ↓
MP4 → MKV
```

---

## Frame

A single image in a video sequence.

For a 30 FPS video:

```text
30 frames ≈ 1 second
```

---

## Frame Rate

Number of video frames displayed per second.

Common values:

```text
24 FPS
25 FPS
30 FPS
60 FPS
```

---

## Timestamp

A temporal position associated with media data.

Examples:

```text
00:00:05.200
00:01:32.750
```

Timestamps are critical for:

* clipping
* synchronization
* subtitles
* transcript segments
* scene detection

---

## Duration

The total temporal length of a media stream or asset.

Example:

```text
duration = 125.4 seconds
```

---

## Resolution

The dimensions of a video frame.

Example:

```text
1920 × 1080
```

---

## Aspect Ratio

The proportional relationship between width and height.

Examples:

```text
16:9
4:3
1:1
9:16
```

---

## Thumbnail

A smaller preview image generated from a media asset.

Typical uses:

* media library
* dashboard
* timeline
* video preview

---

## Clip

A portion of an existing media asset.

Example:

```text
Original:
00:00 → 10:00

Clip:
02:30 → 03:15
```

---

## Scene

A detected logical segment of a video representing a change or coherent visual sequence.

Example:

```text
Scene 1 → 00:00–00:12
Scene 2 → 00:12–00:38
Scene 3 → 00:38–01:05
```

---

## Stream

An individual audio, video, subtitle, or other encoded sequence inside a media container.

---

## Sample Rate

The number of audio samples captured per second.

Examples:

```text
44100 Hz
48000 Hz
```

---

## Channel

An individual audio signal.

Examples:

```text
Mono   → 1 channel
Stereo → 2 channels
```

---

## Downmixing

Combining multiple audio channels into fewer channels.

Example:

```text
5.1 Audio
   ↓
Stereo
```

---

## Resampling

Changing the audio sample rate.

Example:

```text
48000 Hz
   ↓
16000 Hz
```

---

# 4. FFmpeg Terms

## FFmpeg

A multimedia framework providing libraries and tools for processing audio, video, and other multimedia data.

In this project, FFmpeg is integrated primarily through its native libraries.

---

## libavformat

FFmpeg library responsible primarily for:

* container handling
* demuxing
* muxing
* stream discovery

---

## libavcodec

FFmpeg library providing encoding and decoding functionality.

---

## libavutil

FFmpeg utility library containing common multimedia data structures and helper functionality.

---

## libswscale

FFmpeg library used for:

* image scaling
* pixel-format conversion

---

## libswresample

FFmpeg library used for:

* audio resampling
* audio sample-format conversion
* channel conversion

---

## libavfilter

FFmpeg library used for constructing media-processing filter graphs.

---

## Native FFmpeg Integration

Using FFmpeg's C/C++ libraries directly instead of invoking the `ffmpeg` command-line executable as a subprocess.

The project prefers native integration for performance, control, and structured error handling.

---

## FFmpeg Filter Graph

A graph describing a sequence of media transformations.

Example:

```text
Input
  ↓
Scale
  ↓
Format Conversion
  ↓
Output
```

---

# 5. C++ Media Engine Terms

## Media Engine

The C++ processing subsystem responsible for performance-critical media operations.

Responsibilities include:

* probing
* decoding
* thumbnail generation
* clipping
* audio extraction
* media transformation

---

## Worker

A process that executes asynchronous jobs.

Workers are specialized by capability.

Examples:

```text
C++ Media Worker
Python Analytics Worker
```

---

## Worker Capability

A declared operation that a worker can execute.

Example:

```text
probe
thumbnail
clip
audio_extract
```

---

## Worker Registration

The process through which a worker announces itself to the control plane.

Registration may include:

* worker type
* version
* capabilities
* hostname
* resource information

---

## Worker Heartbeat

A periodic signal indicating that a worker is alive.

Example:

```text
Worker
  ↓ heartbeat
Gateway
  ↓
worker.last_seen
```

A worker that stops sending heartbeats can be considered stale.

---

## Worker Lease

A temporary ownership period during which a worker is responsible for a job.

Example:

```text
Job
 ↓
Worker A leases job
 ↓
Worker A processes
```

If the lease expires, another worker may recover the job.

---

## Worker Isolation

Separating media-processing workers from the public API and other infrastructure.

This limits the impact of:

* crashes
* memory exhaustion
* malicious media
* dependency failures

---

# 6. Job and Queue Terms

## Job

A unit of asynchronous work.

Examples:

```text
media_probe
thumbnail_generation
video_clip
audio_extract
transcription
sentiment_analysis
```

---

## Job Queue

A system that stores pending work until a worker is available.

This project uses Redis for initial job coordination.

---

## Queue Producer

A component that creates or publishes jobs.

The Node.js gateway is the primary producer.

---

## Queue Consumer

A worker that receives jobs and executes them.

---

## Job Priority

A value determining the relative scheduling importance of a job.

Example:

```text
HIGH
NORMAL
LOW
```

---

## Job Dependency

A relationship in which one job must complete before another can start.

Example:

```text
Probe
  ↓
Transcription
```

---

## DAG

**Directed Acyclic Graph.**

A graph where dependencies flow in one direction and cycles are not permitted.

Example:

```text
Probe
 ├── Thumbnail
 ├── Audio Extract
 └── Scene Detection
          ↓
     AI Analysis
```

---

## Job State Machine

The formally defined set of valid job states and transitions.

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

Failure path:

```text
running
  ↓
failed
  ↓
retrying
  ↓
queued
```

---

## Job Attempt

One execution attempt of a job.

Example:

```text
Attempt 1 → failed
Attempt 2 → failed
Attempt 3 → completed
```

---

## Retry

A new execution attempt after a recoverable failure.

---

## Backoff

A delay introduced before retrying a failed operation.

Typical strategy:

```text
1s
2s
4s
8s
...
```

---

## Dead Letter

A job that cannot be successfully processed after the permitted retry attempts.

Such jobs require investigation or manual recovery.

---

## Idempotency

The property that repeating the same operation does not produce unintended additional effects.

Example:

```text
Complete Job
Complete Job
Complete Job
```

should still result in one logical completion.

---

## Idempotency Key

A client-provided identifier used to recognize duplicate requests.

Example:

```text
Idempotency-Key: abc123
```

---

## Backpressure

A mechanism for preventing an overloaded component from being overwhelmed by incoming work.

Example:

```text
Too many jobs
     ↓
Queue grows
     ↓
Workers scale or admission is limited
```

---

# 7. Redis Terms

## Redis

An in-memory data platform used in this project primarily for asynchronous job coordination and short-lived state.

---

## Redis Stream

An append-oriented Redis data structure suitable for event and message processing.

---

## Consumer Group

A Redis Streams mechanism that distributes stream messages among multiple consumers.

---

## Acknowledgment

A signal indicating that a consumer has successfully processed a message.

---

## Pending Entry

A queue message delivered to a consumer but not yet acknowledged.

---

## Queue Backlog

The amount of unprocessed work waiting in the queue.

---

# 8. Database Terms

## PostgreSQL

The relational database used as the authoritative metadata and control-plane database.

---

## Source of Truth

The authoritative system from which application state should ultimately be derived.

For durable job and application state:

```text
PostgreSQL = Source of Truth
Redis      = Transport / Coordination
```

---

## Migration

A version-controlled change to the database schema.

Example:

```text
001_initial_schema
002_add_jobs
003_add_results
```

---

## Primary Key

A unique identifier for a database record.

The project primarily uses UUID identifiers.

---

## Foreign Key

A database constraint linking one table to another.

Example:

```text
jobs.media_id
       ↓
media.id
```

---

## Index

A database structure that improves lookup performance.

Indexes should be designed around actual query patterns.

---

## Transaction

A group of database operations executed as one logical unit.

A transaction should preserve:

```text
Atomicity
Consistency
Isolation
Durability
```

---

## ACID

Properties of reliable relational transactions:

* **Atomicity**
* **Consistency**
* **Isolation**
* **Durability**

---

## Soft Delete

Marking a record as deleted rather than physically removing it immediately.

Used for selected media records to support recovery, auditing, and lifecycle management.

---

## Audit Log

A persistent record of security- or business-relevant actions.

Examples:

```text
login
media deletion
permission change
job cancellation
administrative action
```

---

## Outbox Pattern

A reliability pattern in which an event is first stored transactionally in the database and later published to an external messaging system.

Example:

```text
PostgreSQL
    ↓
Outbox Event
    ↓
Publisher
    ↓
Redis
```

This reduces the risk of losing events between database commits and message publication.

---

# 9. Object Storage Terms

## Object Storage

A storage system designed for large binary objects.

Used for:

* original media
* thumbnails
* clips
* extracted audio
* manifests
* analysis artifacts

---

## Object Key

The unique path-like identifier of an object.

Example:

```text
media/123/original/456.mp4
```

---

## Bucket

A logical container for objects in an object-storage system.

---

## Signed URL

A temporary URL granting controlled access to a private object.

Signed URLs should have:

* limited lifetime
* limited permissions
* object-specific scope

---

## Checksum

A value used to verify data integrity.

The project may use SHA-256 for important artifacts.

---

# 10. Python and AI/ML Terms

## Analytics Worker

The Python service responsible for higher-level analysis of media-derived artifacts.

---

## AI/ML Pipeline

A sequence of operations using artificial intelligence or machine-learning models.

Example:

```text
Audio
 ↓
Preprocessing
 ↓
Speech Recognition
 ↓
Transcript
 ↓
Sentiment Analysis
```

---

## Model

A trained machine-learning system used to generate predictions or classifications.

---

## Inference

Running a trained model against input data.

---

## Preprocessing

Transforming raw data into the format required by a model.

---

## Postprocessing

Transforming model output into a normalized application representation.

---

## Confidence Score

A numerical estimate associated with a model prediction.

The project normalizes confidence values to:

```text
0.0 ≤ confidence ≤ 1.0
```

---

## Transcription

Converting spoken audio into text.

---

## Transcript

The resulting textual representation of spoken content.

---

## Transcript Segment

A time-bounded portion of a transcript.

Example:

```text
00:12.5 → 00:15.2
"Hello everyone."
```

---

## Speaker Diarization

Determining which speaker is active during different portions of an audio recording.

Example:

```text
Speaker 1 → 00:00–00:12
Speaker 2 → 00:12–00:25
```

This may be added as an advanced capability.

---

## Sentiment Analysis

Estimating the emotional or sentiment orientation of text.

Example:

```text
Positive
Neutral
Negative
```

---

## Object Detection

Identifying objects within an image or video frame and locating them spatially.

---

## Bounding Box

A rectangular region identifying the location of a detected object.

Example:

```text
x
y
width
height
confidence
label
```

---

## Face Detection

Detecting human faces in images or video frames.

Face detection does not inherently mean identifying the person.

---

## Scene Detection

Identifying meaningful transitions or changes between video scenes.

---

# 11. Manifest Terms

## Processing Manifest

A machine-readable description of the inputs and artifacts associated with a processing stage.

Example:

```json
{
  "manifestVersion": 1,
  "mediaId": "uuid",
  "jobId": "uuid",
  "artifacts": []
}
```

---

## Manifest Version

The schema version of a processing manifest.

Example:

```text
manifestVersion = 1
```

Workers must explicitly support the versions they understand.

---

## Schema Compatibility

The ability of one component to correctly consume data produced by another component.

---

## Artifact Integrity

The guarantee that an artifact has not been unexpectedly modified or corrupted.

Checksums can be used to validate integrity.

---

# 12. API and Web Terms

## REST

**Representational State Transfer.**

An architectural style commonly used for HTTP APIs.

The project's public API uses REST-style endpoints.

---

## Endpoint

A specific API operation exposed at a URL and HTTP method.

Example:

```text
GET /api/v1/media
```

---

## HTTP Method

The operation being performed against an HTTP resource.

Common methods:

```text
GET
POST
PUT
PATCH
DELETE
```

---

## HTTP Status Code

A numeric response indicating the result of an HTTP request.

Common project codes:

```text
200 OK
201 Created
202 Accepted
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Entity
429 Too Many Requests
500 Internal Server Error
503 Service Unavailable
```

---

## API Versioning

Maintaining explicit versions of the public API.

Example:

```text
/api/v1/...
```

---

## Pagination

Returning large result sets in smaller portions.

Example:

```text
page=1
pageSize=50
```

or cursor-based pagination.

---

## Rate Limiting

Restricting how frequently a client may perform requests.

Used to reduce:

* abuse
* accidental overload
* brute-force attempts
* resource exhaustion

---

## WebSocket

A persistent bidirectional communication protocol used for real-time updates.

The project uses WebSockets for job progress and system events.

---

## WebSocket Event

A structured message delivered to connected clients.

Examples:

```text
job.started
job.progress
job.completed
job.failed
```

---

## Reconnection

The process through which a client restores a WebSocket connection after disconnection.

---

# 13. Authentication and Security Terms

## Authentication

Verifying who a user or service is.

---

## Authorization

Determining what an authenticated identity is allowed to access or perform.

---

## RBAC

**Role-Based Access Control.**

Permissions are associated with roles rather than individual users.

Example:

```text
User
Admin
Worker
```

---

## IDOR

**Insecure Direct Object Reference.**

A vulnerability where a user can access another user's resource by changing an identifier.

Example:

```text
/api/v1/media/user-A-media
```

being changed to:

```text
/api/v1/media/user-B-media
```

The API must prevent unauthorized access.

---

## JWT

**JSON Web Token.**

A signed token format commonly used for transmitting authentication claims.

---

## Access Token

A short-lived credential used to access protected APIs.

---

## Refresh Token

A longer-lived credential used to obtain a new access token.

---

## Session

A server-managed representation of an authenticated login.

---

## CORS

**Cross-Origin Resource Sharing.**

A browser security mechanism controlling which origins may access a web API.

---

## CSRF

**Cross-Site Request Forgery.**

An attack where a user's authenticated browser is tricked into performing an unintended action.

---

## SSRF

**Server-Side Request Forgery.**

An attack in which an attacker causes a server to make unintended network requests.

---

## SQL Injection

An attack where malicious SQL syntax is inserted into database queries.

The project prevents this through parameterized queries and strict input validation.

---

## Path Traversal

An attack attempting to access unintended filesystem locations.

Typical pattern:

```text
../
```

---

## Command Injection

An attack where untrusted input is interpreted as operating-system commands.

The media engine must not construct shell commands from untrusted user input.

---

## Least Privilege

Granting each user, service, and worker only the permissions it actually requires.

---

## Defense in Depth

Using multiple independent security controls rather than relying on one protection.

---

## Trust Boundary

A point where data crosses from one security context into another.

Example:

```text
Internet
   ↓
API Gateway
```

is a major trust boundary.

---

# 14. Reliability Terms

## Fault Tolerance

The ability of a system to continue operating despite component failures.

---

## Failure Domain

A component or infrastructure boundary within which failures may occur.

Example:

```text
C++ Worker A
```

should fail without bringing down the API gateway.

---

## Graceful Shutdown

A controlled process termination that allows the service to:

* stop accepting new work
* finish safe operations
* release leases
* flush logs
* close connections

---

## Recovery

Returning the system to a valid operational state after failure.

---

## Retryable Error

An error expected to potentially succeed if attempted again.

Examples:

```text
network timeout
temporary storage failure
temporary provider outage
```

---

## Permanent Error

An error unlikely to succeed through retrying.

Examples:

```text
corrupt media
unsupported codec
invalid manifest
```

---

## Circuit Breaker

A reliability pattern that temporarily stops calls to a failing dependency.

This may be introduced as the platform grows.

---

## Timeout

A maximum period allowed for an operation before it is considered failed.

---

## Health Check

An endpoint or mechanism used to determine service health.

---

## Liveness

Indicates whether a process is alive.

---

## Readiness

Indicates whether a process is capable of serving requests or performing work.

---

# 15. Observability Terms

## Observability

The ability to understand internal system behavior from external outputs.

Primary signals:

```text
Logs
Metrics
Traces
```

---

## Structured Logging

Logging events as structured data rather than arbitrary text.

Example:

```json
{
  "level": "info",
  "event": "job.completed",
  "jobId": "uuid",
  "durationMs": 1250
}
```

---

## Correlation ID

An identifier connecting related operations across services.

Example:

```text
API
 ↓ correlationId
Job
 ↓ correlationId
C++
 ↓ correlationId
Python
```

---

## Request ID

An identifier associated with a specific API request.

---

## Trace

A representation of an operation across multiple distributed components.

---

## Span

One timed operation within a distributed trace.

Example:

```text
HTTP Request
 ├── DB Query
 ├── Redis Publish
 └── Worker Processing
```

---

## Metric

A numerical measurement representing system behavior.

Examples:

```text
job_processing_seconds
queue_depth
api_request_duration
worker_cpu_usage
```

---

## P50

The 50th percentile.

Approximately half of observations are below this value.

---

## P95

The 95th percentile.

Approximately 95% of observations are below this value.

---

## P99

The 99th percentile.

Useful for identifying high-latency tail behavior.

---

# 16. Performance Terms

## Throughput

The amount of work processed per unit of time.

Example:

```text
100 media jobs/minute
```

---

## Latency

The time required to complete an operation.

---

## Queue Latency

Time between job submission and worker execution.

---

## Processing Latency

Time required for a worker to process a job.

---

## End-to-End Latency

Time from the user's initial request until the complete result is available.

---

## CPU Utilization

Percentage of CPU resources consumed by a process or system.

---

## Memory Footprint

Amount of memory consumed by a process.

---

## Peak RSS

**Peak Resident Set Size.**

The maximum amount of physical memory occupied by a process during execution.

---

## Benchmark

A repeatable performance measurement.

---

## Load Test

Testing under expected or specified workload.

---

## Stress Test

Testing beyond normal operating limits to determine system behavior under extreme load.

---

## Soak Test

Running a system under sustained workload for an extended period.

Used to discover:

* memory leaks
* resource leaks
* gradual performance degradation

---

## Scalability

The ability of a system to handle increasing workload by adding resources or improving efficiency.

---

## Horizontal Scaling

Adding more instances or workers.

```text
1 Worker
   ↓
10 Workers
   ↓
100 Workers
```

---

## Vertical Scaling

Increasing the resources of a single machine.

Example:

```text
4 CPU / 8 GB RAM
        ↓
16 CPU / 32 GB RAM
```

---

# 17. Testing Terms

## Unit Test

Tests a small isolated unit of code.

---

## Integration Test

Tests interaction between multiple components.

---

## Contract Test

Tests whether two components continue to follow a shared interface contract.

---

## End-to-End Test

Tests a complete user or business workflow across the system.

---

## Golden Path

The primary successful workflow.

For this project:

```text
Upload
 ↓
Process
 ↓
Analyze
 ↓
Display
```

---

## Regression Test

A test designed to ensure that a previously fixed defect does not return.

---

## Fixture

A controlled input used by tests.

Examples:

```text
video-h264.mp4
valid-manifest.json
test-user.json
```

---

## Golden File

A known expected output used to compare generated results.

---

## Test Corpus

A collection of representative test inputs.

The project's media corpus contains different codecs, containers, resolutions, durations, and malformed files.

---

## Fuzz Testing

Providing large numbers of unexpected or malformed inputs to find crashes, security issues, and undefined behavior.

---

## Property-Based Testing

Testing general properties over many automatically generated inputs rather than testing only predefined examples.

---

## Test Coverage

A measurement of how much code or behavior is exercised by tests.

Coverage may include:

* line coverage
* branch coverage
* function coverage

---

## Flaky Test

A test that sometimes passes and sometimes fails without a relevant code change.

Flaky tests are treated as defects.

---

## Test Isolation

Ensuring that tests do not depend on execution order or shared mutable state.

---

## Acceptance Criteria

Conditions that must be satisfied for a feature or release to be considered complete.

---

# 18. CI/CD Terms

## CI

**Continuous Integration.**

Automatically validating changes when code is pushed or submitted.

---

## CD

**Continuous Delivery / Continuous Deployment.**

Automating the process of preparing or deploying validated software.

---

## Pipeline

A sequence of automated development or deployment stages.

Example:

```text
Lint
 ↓
Unit
 ↓
Integration
 ↓
Security
 ↓
Build
 ↓
E2E
 ↓
Release
```

---

## Build Artifact

A generated output of the build process.

Examples:

```text
binary
Docker image
JavaScript bundle
Python package
```

---

## SBOM

**Software Bill of Materials.**

A machine-readable inventory of software dependencies contained in a project or artifact.

---

## SAST

**Static Application Security Testing.**

Security analysis performed against source code or compiled representations without executing the application normally.

---

## SCA

**Software Composition Analysis.**

Scanning dependencies for known vulnerabilities and licensing information.

---

## Container Image

A packaged filesystem and metadata used to run an application in a container.

---

# 19. Docker and Deployment Terms

## Docker

A containerization platform used to package and run application services consistently.

---

## Container

An isolated runtime environment for an application process.

---

## Docker Compose

A tool for defining and running multiple related containers.

The project uses it for local and production-like development.

---

## Service

A logical application or infrastructure component.

Examples:

```text
gateway
postgres
redis
cpp-worker
python-worker
frontend
```

---

## Healthcheck

A container-level command used to determine whether a service is functioning.

---

## Environment Variable

A runtime configuration value provided outside application source code.

Example:

```text
DATABASE_URL
REDIS_URL
OBJECT_STORAGE_ENDPOINT
```

---

## Secret

Sensitive configuration data such as:

```text
password
API key
private key
token
```

Secrets must not be committed to source control.

---

## Infrastructure as Code

Defining infrastructure configuration in version-controlled files.

---

# 20. Development Terms

## Repository

The version-controlled project source tree.

---

## Monorepo

A repository containing multiple related applications or services.

This project follows a repository structure containing:

```text
gateway-node
media-engine-cpp
analytics-python
frontend-react
```

---

## Module

A logically isolated unit of functionality inside a service.

---

## Adapter

A component translating between an internal interface and an external system.

Examples:

```text
S3 Adapter
AI Provider Adapter
Redis Adapter
```

---

## Provider

An implementation of a service dependency.

Example:

```text
OpenAI-compatible provider
Local model provider
Mock provider
```

---

## Dependency Injection

Providing dependencies to a component rather than constructing them internally.

Useful for:

* testing
* configuration
* provider replacement

---

## Configuration

Runtime settings controlling application behavior.

Configuration should be separated from source code.

---

## Feature Flag

A runtime-controlled switch used to enable or disable functionality.

---

## Semantic Versioning

A versioning convention:

```text
MAJOR.MINOR.PATCH
```

Example:

```text
1.4.2
```

---

# 21. Git and Engineering Terms

## Commit

A version-controlled snapshot of changes.

---

## Branch

An independent line of development.

---

## Pull Request

A proposed change reviewed before being merged.

---

## ADR

**Architecture Decision Record.**

A document capturing an important architectural decision, its context, alternatives, and consequences.

Project ADRs include:

```text
ADR-001-polyglot-architecture
ADR-002-ffmpeg-native-integration
ADR-003-redis-job-queue
ADR-004-object-storage
ADR-005-postgresql
ADR-006-worker-isolation
```

---

## Technical Debt

The future cost created by deliberately or accidentally choosing a less-than-ideal implementation.

---

## Definition of Done

A defined checklist that determines whether a task is actually complete.

---

# 22. Data Lifecycle Terms

## Upload

The process of transferring user media into the platform.

---

## Registration

Creating metadata representing a media resource in PostgreSQL.

---

## Processing

Executing one or more jobs against a media asset.

---

## Transformation

Changing the representation of media.

Examples:

```text
resize
transcode
clip
extract audio
```

---

## Retention

The period for which data is kept.

---

## Data Deletion

Removing or logically deleting user media and associated artifacts according to lifecycle policy.

---

## Garbage Collection

Removing temporary or orphaned artifacts that are no longer required.

---

## Reproducibility

The ability to repeat a processing operation and understand or reproduce its output using recorded inputs, versions, configuration, and metadata.

---

# 23. Platform-Specific Terms

## Media Studio

The React-based user interface for interacting with media.

Expected capabilities include:

* upload
* preview
* timeline
* transcript
* detections
* clips
* AI insights
* job progress

---

## Processing Pipeline

The complete sequence of operations applied to uploaded media.

Example:

```text
Upload
 ↓
Probe
 ↓
Thumbnail
 ↓
Audio Extraction
 ↓
Transcription
 ↓
Detection
 ↓
AI Analysis
 ↓
Results
```

---

## Processing Stage

One logical step within the processing pipeline.

Examples:

```text
probe
thumbnail
transcription
detection
sentiment
```

---

## Processing Result

Structured information produced by a processing job.

---

## Insight

A higher-level interpretation derived from processed media.

Examples:

* sentiment
* topic
* summary
* detected entities
* important moments

---

## Media Intelligence

The overall capability of extracting useful machine-readable information from media.

---

# 24. Project Acronyms

| Acronym   | Meaning                                           |
| --------- | ------------------------------------------------- |
| ADR       | Architecture Decision Record                      |
| AI        | Artificial Intelligence                           |
| API       | Application Programming Interface                 |
| CI        | Continuous Integration                            |
| CD        | Continuous Delivery / Deployment                  |
| CORS      | Cross-Origin Resource Sharing                     |
| CSRF      | Cross-Site Request Forgery                        |
| CPU       | Central Processing Unit                           |
| DAG       | Directed Acyclic Graph                            |
| E2E       | End-to-End                                        |
| FFmpeg    | Multimedia processing framework                   |
| FPS       | Frames Per Second                                 |
| GPU       | Graphics Processing Unit                          |
| HTTP      | Hypertext Transfer Protocol                       |
| IDOR      | Insecure Direct Object Reference                  |
| JWT       | JSON Web Token                                    |
| ML        | Machine Learning                                  |
| MIME      | Multipurpose Internet Mail Extensions             |
| MVP       | Minimum Viable Product                            |
| P50       | 50th percentile                                   |
| P95       | 95th percentile                                   |
| P99       | 99th percentile                                   |
| REST      | Representational State Transfer                   |
| RBAC      | Role-Based Access Control                         |
| RSS       | Resident Set Size                                 |
| SAST      | Static Application Security Testing               |
| SCA       | Software Composition Analysis                     |
| SBOM      | Software Bill of Materials                        |
| SQL       | Structured Query Language                         |
| SSRF      | Server-Side Request Forgery                       |
| TLS       | Transport Layer Security                          |
| UUID      | Universally Unique Identifier                     |
| UX        | User Experience                                   |
| WebSocket | Full-duplex persistent web communication protocol |

---

# 25. Important Terminology Distinctions

## Authentication vs Authorization

```text
Authentication
= Who are you?

Authorization
= What are you allowed to do?
```

---

## Container vs Codec

```text
Container
= How streams are packaged

Codec
= How media is encoded/decoded
```

Example:

```text
MP4
 ├── H.264 video
 └── AAC audio
```

---

## Job vs Worker

```text
Job
= Work that needs to be performed

Worker
= Process that performs the work
```

---

## Artifact vs Result

```text
Artifact
= Physical/generated file

Result
= Structured metadata describing processing output
```

---

## Queue vs Database

```text
Redis
= Fast job transport/coordination

PostgreSQL
= Durable application state/source of truth
```

---

## Clip vs Asset

```text
Asset
= Stored media object

Clip
= Logical or rendered portion of media
```

---

## Scene vs Clip

```text
Scene
= Automatically detected semantic/video segment

Clip
= User-requested or system-generated extract
```

---

## Decode vs Transcode

```text
Decode
= Encoded → raw representation

Transcode
= Decode → process → encode
```

---

## Horizontal vs Vertical Scaling

```text
Horizontal
= Add more machines/processes

Vertical
= Add more resources to one machine
```

---

# 26. Canonical Project Vocabulary

The following terms should be preferred consistently throughout the repository.

| Preferred Term      | Avoid Ambiguous Alternatives |
| ------------------- | ---------------------------- |
| Media               | File                         |
| Media Asset         | File object                  |
| Artifact            | Output file                  |
| Job                 | Task/request                 |
| Worker              | Processor                    |
| Media Engine        | Video processor              |
| Processing Manifest | Metadata JSON                |
| Object Storage      | File storage                 |
| Gateway             | Backend server               |
| Analytics Worker    | AI server                    |
| Job Lease           | Lock                         |
| Job Attempt         | Retry count                  |
| Processing Result   | Output                       |
| Correlation ID      | Tracking ID                  |
| Media Studio        | Frontend                     |
| Processing Pipeline | Workflow                     |

The goal is to make architecture, code, API documentation, and operational terminology consistent.

---

# 27. Lifecycle Vocabulary

The canonical media lifecycle is:

```text
Registered
    ↓
Uploaded
    ↓
Validated
    ↓
Queued
    ↓
Processing
    ↓
Analyzed
    ↓
Completed
    ↓
Available
```

Failure:

```text
Processing
    ↓
Failed
    ↓
Retrying
    ↓
Queued
```

Permanent failure:

```text
Failed
   ↓
Dead Letter / Terminal Failure
```

---

# 28. Job Lifecycle Vocabulary

Canonical states:

```text
queued
leased
running
completed
failed
retrying
cancelled
```

Canonical concepts:

```text
Job
Job Attempt
Job Lease
Job Dependency
Job Event
Job Result
```

These terms should be used consistently in:

* PostgreSQL
* Redis
* Node.js
* C++
* Python
* React
* API documentation

---

# 29. Worker Lifecycle Vocabulary

Canonical lifecycle:

```text
starting
   ↓
registering
   ↓
ready
   ↓
busy
   ↓
idle
   ↓
stopping
   ↓
offline
```

Workers should expose:

* identity
* version
* capabilities
* heartbeat
* current job
* health state

---

# 30. Media Pipeline Vocabulary

Canonical pipeline:

```text
Original Media
      ↓
Validation
      ↓
Probe
      ↓
Metadata
      ↓
C++ Processing
      ↓
Artifacts
      ↓
Manifest
      ↓
Python Analytics
      ↓
Structured Results
      ↓
PostgreSQL
      ↓
React Media Studio
```

---

# 31. Security Vocabulary

Security-critical concepts include:

```text
Authentication
Authorization
Least Privilege
Trust Boundary
Worker Isolation
Input Validation
Media Validation
Object Ownership
Secret Management
Audit Logging
Rate Limiting
Resource Limits
Sandboxing
```

Uploaded media must always be treated as untrusted input.

---

# 32. Testing Vocabulary

The canonical testing hierarchy is:

```text
Unit
 ↓
Contract
 ↓
Integration
 ↓
E2E
 ↓
Acceptance
```

Additional testing dimensions:

```text
Security
Performance
Concurrency
Failure Injection
Fuzzing
Memory Safety
Load
Stress
Soak
```

---

# 33. Architecture Vocabulary

The canonical architecture terminology is:

```text
Frontend
    ↓
API Gateway
    ↓
Control Plane
    ↓
Queue
    ↓
Worker
    ↓
Data Plane
    ↓
Artifact
    ↓
Analytics
    ↓
Result
```

This vocabulary should be used in architecture diagrams and technical discussions.

---

# 34. Glossary Maintenance

This document must be updated when:

* a new major component is introduced
* an architectural term changes
* a new processing stage is introduced
* a new API concept is introduced
* a new security concept becomes relevant
* an existing term becomes ambiguous
* a project-specific acronym is introduced

Terminology changes should preferably be recorded through an ADR when they represent an architectural change.

---

# 35. Documentation Cross-Reference

This glossary provides terminology for:

```text
01-product-requirements.md
        ↓
02-architecture.md
        ↓
03-data-model.md
        ↓
04-api-reference.md
        ↓
05-roadmap-and-phases.md
        ↓
06-development-guide.md
        ↓
07-security.md
        ↓
08-gap-analysis.md
        ↓
09-testing-strategy.md
        ↓
10-glossary.md
```

The glossary therefore acts as the common vocabulary layer for the entire documentation set.

---

# 36. Final Terminology Principle

The project should maintain a simple rule:

> **One concept should have one canonical name.**

For example:

```text
Job
not sometimes Task

Worker
not sometimes Processor

Artifact
not sometimes Output File

Media Asset
not sometimes File

Processing Manifest
not sometimes Metadata JSON
```

Consistent terminology reduces ambiguity between:

* developers
* APIs
* database schemas
* workers
* documentation
* tests
* operations
* future contributors

The glossary is therefore not merely a dictionary; it is part of the platform's engineering contract.

---

# 37. Final Canonical Model

```text
                         USER
                           │
                           ▼
                    ┌─────────────┐
                    │ Media Studio│
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ API Gateway │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         PostgreSQL      Redis      Object Storage
              │            │            │
              │            ▼            │
              │      ┌──────────┐       │
              │      │  Queue   │       │
              │      └────┬─────┘       │
              │           │             │
              │     ┌─────┴─────┐       │
              │     ▼           ▼       │
              │    C++        Python    │
              │   Worker      Worker    │
              │     │           │       │
              │     └─────┬─────┘       │
              │           ▼             │
              │      Artifacts          │
              │           │             │
              └───────────┼─────────────┘
                          ▼
                    Processing Results
                          │
                          ▼
                    Media Studio
```

The platform's core vocabulary can therefore be summarized as:

```text
Media
  ↓
Asset
  ↓
Job
  ↓
Queue
  ↓
Worker
  ↓
Processing
  ↓
Artifact
  ↓
Manifest
  ↓
Analytics
  ↓
Result
  ↓
Insight
  ↓
Media Studio
```

This vocabulary forms the canonical terminology baseline for the High-Performance Distributed Media Analytics Platform.
