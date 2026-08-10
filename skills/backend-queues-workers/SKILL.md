````md
---
name: backend-queues-workers
description: Design and implement reliable background job processing using queues and workers for long-running, expensive, asynchronous, bursty, or retryable backend operations. Use this skill when backend work should be decoupled from HTTP requests, when workloads require controlled concurrency, retries, scheduling, prioritization, batching, delayed execution, progress tracking, or protection against traffic spikes. Use queues and workers only when they provide a real architectural benefit; do not introduce them into simple CRUD operations unnecessarily.
---

# Backend Queues and Workers

Design backend background processing so expensive or asynchronous work can be handled reliably without unnecessarily blocking user-facing API requests.

Queues and workers should be used to separate:

```text
Request handling
````

from:

```text
Background processing
```

when the operation does not need to finish before the API can respond.

The objective is to create backend systems that can:

* remain responsive during expensive operations
* absorb temporary traffic spikes
* process workloads asynchronously
* control concurrency
* retry transient failures safely
* prevent one expensive task from blocking unrelated requests
* process large workloads in manageable batches
* provide job status when users need progress information
* recover from worker failures
* avoid duplicate processing
* prevent retry storms
* provide visibility into queue health
* scale workers independently when necessary

Do not introduce a queue simply because a project is expected to become "scalable."

A simple application may only need:

```text
Client
  ↓
API
  ↓
Database
```

A system with expensive asynchronous work may benefit from:

```text
Client
  ↓
API
  ↓
Queue
  ↓
Worker
  ↓
Database / External Service
```

Choose the simplest architecture that satisfies the project's actual requirements.

---

# Core Principles

## Separate User-Facing Work from Background Work

The HTTP request lifecycle should remain focused on work that the user actually needs completed before receiving the response.

Avoid unnecessarily keeping requests open while performing operations such as:

* report generation
* large exports
* document processing
* image processing
* sending large batches of emails
* notification fan-out
* data synchronization
* large imports
* file conversion
* expensive calculations
* third-party API synchronization
* scheduled maintenance
* bulk database operations

Instead of:

```text
POST /reports
     ↓
Generate report
     ↓
Process thousands of records
     ↓
Upload file
     ↓
Send notification
     ↓
Return response
```

prefer:

```text
POST /reports
     ↓
Create job
     ↓
Queue job
     ↓
Return job ID
     ↓
Worker processes job
     ↓
Store result
     ↓
Notify client
```

The API should remain responsive while the worker performs the expensive work.

---

# When to Use a Queue

Use queues when at least one of the following is true:

* the operation is long-running
* the operation is CPU-intensive
* the operation is I/O-heavy
* the operation can happen after the HTTP response
* the operation can tolerate asynchronous execution
* traffic can arrive in bursts
* work needs controlled concurrency
* failed operations need retrying
* jobs need delayed execution
* jobs need scheduling
* jobs need prioritization
* workers need to scale independently
* processing should be isolated from user-facing requests

Examples:

```text
Generate monthly report
Process uploaded document
Send 10,000 notifications
Synchronize external records
Resize uploaded images
Generate spreadsheet
Import thousands of records
Recalculate large dataset
Process payment webhook asynchronously
```

---

# When NOT to Use a Queue

Do not introduce a queue for operations that are:

* extremely fast
* required immediately
* simple CRUD operations
* simple database reads
* simple database updates
* authentication checks
* basic validation
* normal page data retrieval

For example:

```text
GET /drivers/:id
```

does not normally need:

```text
API
 ↓
Queue
 ↓
Worker
 ↓
Database
 ↓
API
```

That would add unnecessary complexity and latency.

Prefer:

```text
API
 ↓
Database
 ↓
Response
```

---

# Synchronous vs Asynchronous Work

## Synchronous

Use synchronous processing when the client needs the result immediately.

```text
Client
 ↓
API
 ↓
Process
 ↓
Response
```

Example:

```text
GET /drivers/123
```

---

## Asynchronous

Use asynchronous processing when the work can continue after the initial request.

```text
Client
 ↓
API
 ↓
Create job
 ↓
Queue
 ↓
202 Accepted
```

Then:

```text
Queue
 ↓
Worker
 ↓
Process
 ↓
Store result
```

The client can later retrieve the result or receive an update.

---

# HTTP Response for Queued Work

When a request successfully creates a background job, consider returning:

```text
202 Accepted
```

rather than pretending the work has already completed.

A response may contain:

```json
{
  "success": true,
  "message": "Job accepted for processing",
  "jobId": "..."
}
```

The exact response structure should follow the project's API conventions.

Do not return:

```text
success: true
```

if the actual background operation has not yet succeeded.

The job was accepted, not necessarily completed.

---

# Job Lifecycle

Define an explicit lifecycle for jobs.

A typical lifecycle is:

```text
QUEUED
   ↓
PROCESSING
   ↓
COMPLETED
```

or:

```text
QUEUED
   ↓
PROCESSING
   ↓
FAILED
```

Additional states may include:

```text
WAITING
RETRYING
CANCELLED
DELAYED
PAUSED
EXPIRED
```

Only introduce states that the application actually needs.

---

# Job State Machine

Treat job states as deliberate transitions.

Example:

```text
QUEUED
  ↓
PROCESSING
  ├──→ COMPLETED
  ├──→ FAILED
  └──→ RETRYING
          ↓
      PROCESSING
```

Do not allow arbitrary state changes.

For example, avoid allowing:

```text
COMPLETED
   ↓
QUEUED
```

unless the system explicitly supports reprocessing.

---

# Job Data

A job should contain enough information for the worker to perform its work.

Prefer storing references:

```json
{
  "jobId": "...",
  "type": "GENERATE_REPORT",
  "userId": "...",
  "reportId": "..."
}
```

rather than placing extremely large objects directly into the queue.

---

# Keep Job Payloads Small

Avoid putting:

* large database documents
* huge arrays
* entire files
* large binary data
* unnecessary duplicated information

inside queue messages.

Prefer:

```text
job
 ↓
record ID
 ↓
worker fetches required data
```

instead of:

```text
job
 ↓
entire database record
 ↓
huge embedded payload
```

Small job payloads reduce:

* queue storage
* serialization cost
* network traffic
* memory usage

---

# Job Idempotency

Workers must assume a job may be delivered or executed more than once.

A job should ideally be safe to repeat.

For example:

```text
Process transaction 123
```

should not accidentally create:

```text
Transaction 123
Transaction 123 duplicate
Transaction 123 duplicate
```

because the worker was retried.

---

# Idempotency Keys

For operations where duplication would be harmful, use an idempotency mechanism.

Example:

```text
jobId
operationId
transactionId
externalReference
```

The worker checks whether the operation has already been completed.

Example:

```text
Receive job
 ↓
Check operation ID
 ↓
Already completed?
 ├── Yes → stop safely
 └── No → process
```

---

# At-Least-Once Processing

Many queue systems provide behavior where a job may be delivered more than once.

Design workers accordingly.

Do not assume:

```text
one job
=
exactly one execution
```

unless the queue infrastructure explicitly guarantees that behavior and the application can rely on it.

The safer assumption is:

> A background job may be processed more than once.

---

# Exactly-Once Processing

Be careful when claiming that a job is "exactly once."

Exactly-once behavior is difficult across:

```text
Queue
+
Database
+
External Service
```

For example:

```text
Worker
 ↓
External API succeeds
 ↓
Worker crashes before marking job complete
 ↓
Job is retried
 ↓
External API called again
```

The system may perform the external operation twice.

Use idempotency and transactional patterns where possible instead of assuming exactly-once execution.

---

# Queue Backpressure

Queues provide a buffer when incoming work temporarily exceeds processing capacity.

Example:

```text
Incoming jobs:
████████████████████████

Worker capacity:
████████
```

The queue absorbs the difference.

However, a queue does not eliminate overload.

If:

```text
incoming rate > processing rate
```

for a sustained period, the queue will continue growing.

---

# Queue Depth

Monitor:

```text
number of queued jobs
```

A continuously increasing queue is a warning sign.

Possible causes:

* insufficient workers
* slow processing
* external service slowdown
* database bottleneck
* excessive traffic
* retry storms
* stuck workers

Do not solve queue growth simply by increasing queue capacity.

---

# Queue Age

Queue depth alone is not enough.

Also monitor:

```text
age of oldest waiting job
```

For example:

```text
Queue:
500 jobs

Oldest job:
2 seconds
```

may be healthier than:

```text
Queue:
50 jobs

Oldest job:
45 minutes
```

Queue latency matters.

---

# Worker Concurrency

A worker may process multiple jobs concurrently.

For example:

```text
Worker
 ├── Job A
 ├── Job B
 ├── Job C
 └── Job D
```

Concurrency can increase throughput.

However, too much concurrency can overwhelm:

* database connections
* CPU
* memory
* external APIs
* network
* file systems

Use controlled concurrency.

---

# Worker Concurrency Limits

Avoid unlimited parallel processing.

Bad:

```text
10000 jobs
 ↓
10000 simultaneous operations
```

Prefer:

```text
10000 jobs
 ↓
10 concurrent jobs
 ↓
next jobs
```

The correct value depends on the workload.

---

# CPU-Bound Workers

CPU-heavy jobs require special consideration.

Examples:

* image processing
* video processing
* encryption
* large calculations
* document conversion
* data transformation

Do not allow CPU-heavy work to block the main Node.js event loop.

Consider:

```text
Queue
 ↓
Worker process
 ↓
CPU-heavy operation
```

or:

```text
Queue
 ↓
Worker thread / dedicated worker
```

depending on the workload.

---

# I/O-Bound Workers

I/O-heavy jobs may benefit from concurrency.

Examples:

* external API calls
* database reads
* file operations
* cloud storage operations

However, concurrency should still be bounded.

More simultaneous requests are not always faster.

---

# Worker Isolation

Separate expensive workloads when necessary.

For example:

```text
General jobs
     ↓
General workers

Large report jobs
     ↓
Report workers
```

This prevents a large workload from consuming all worker capacity.

---

# Bulkhead Pattern

Use separate worker pools when one workload could negatively affect another.

Example:

```text
                 Queue
                   │
          ┌────────┴────────┐
          ▼                 ▼
     Normal Jobs       Heavy Jobs
          │                 │
     Workers × 5       Workers × 2
```

A heavy report generation job should not necessarily prevent ordinary notifications from being processed.

---

# Job Priorities

Some workloads are more important than others.

For example:

```text
HIGH
security notification

NORMAL
regular notification

LOW
analytics aggregation
```

Use priorities when the business actually requires them.

Do not create dozens of priority levels.

Keep priority categories understandable.

---

# Fairness

Priority systems can create starvation.

For example:

```text
HIGH jobs
HIGH jobs
HIGH jobs
HIGH jobs
HIGH jobs
...
```

could prevent:

```text
LOW job
```

from ever running.

If low-priority jobs must eventually execute, design the queue strategy accordingly.

---

# Delayed Jobs

Queues can be useful when work should happen later.

Examples:

```text
send reminder in 24 hours
retry after 30 seconds
process after scheduled time
expire temporary record
```

Do not implement delayed jobs using:

```js
setTimeout(...)
```

inside a web server for important persistent work.

Server restarts can lose in-memory timers.

Use a persistent scheduling mechanism when the job matters.

---

# Scheduled Jobs

For recurring work:

```text
Every hour
Every day
Every Monday
```

use a scheduler that creates jobs.

Prefer:

```text
Scheduler
 ↓
Queue
 ↓
Worker
```

rather than putting the entire workload inside the scheduler.

The scheduler should generally trigger work, while workers perform the work.

---

# Retry Strategy

Retries should be deliberate.

A failed job should not automatically retry forever.

Define:

```text
maximum attempts
retryable errors
backoff
jitter
```

---

# Retryable Errors

Retry transient failures such as:

```text
temporary network failure
timeout
temporary service unavailable
temporary database connectivity issue
rate limit
```

Do not normally retry permanent failures such as:

```text
invalid input
missing required record
invalid credentials
permission denied
malformed request
```

Retrying permanent failures wastes resources.

---

# Exponential Backoff

Avoid immediate repeated retries.

Prefer:

```text
Attempt 1
 ↓
100ms

Attempt 2
 ↓
200ms

Attempt 3
 ↓
400ms

Attempt 4
 ↓
800ms
```

The exact timing depends on the application.

---

# Jitter

Multiple workers can accidentally retry at the same time.

Without jitter:

```text
Worker 1 → retry at 1s
Worker 2 → retry at 1s
Worker 3 → retry at 1s
Worker 4 → retry at 1s
```

With jitter:

```text
Worker 1 → retry at 0.8s
Worker 2 → retry at 1.2s
Worker 3 → retry at 0.9s
Worker 4 → retry at 1.4s
```

This reduces synchronized retry spikes.

---

# Retry Storms

Avoid:

```text
External service fails
 ↓
100 jobs fail
 ↓
100 jobs retry immediately
 ↓
service receives another 100 requests
 ↓
service remains overloaded
 ↓
100 jobs retry again
```

Use:

* exponential backoff
* jitter
* retry limits
* rate limits
* circuit breakers where appropriate

---

# Dead-Letter Jobs

Jobs that repeatedly fail should eventually stop retrying.

A dead-letter mechanism can preserve failed jobs for investigation.

Example:

```text
QUEUED
 ↓
PROCESSING
 ↓
FAILED
 ↓
RETRY
 ↓
FAILED
 ↓
RETRY
 ↓
FAILED
 ↓
DEAD LETTER
```

Dead-letter jobs should be visible to administrators or operators when appropriate.

---

# Dead-Letter Handling

A dead-letter job should contain enough information to determine:

```text
What job failed?
Why did it fail?
How many attempts occurred?
When did it fail?
What data was involved?
```

Do not silently discard permanently failed jobs.

---

# Poison Jobs

A poison job is a job that repeatedly causes failures.

Examples:

```text
corrupt file
invalid database reference
unexpected data format
unsupported document
```

Without proper limits, it can consume worker capacity indefinitely.

Use:

```text
retry limit
dead-letter handling
error classification
```

to isolate it.

---

# Job Timeouts

Every potentially long-running job should have a reasonable execution limit when appropriate.

Example:

```text
Report generation
maximum runtime = 5 minutes
```

If a worker gets stuck, the job should not remain active forever.

---

# Worker Crashes

Workers can crash.

The system should be able to recover.

Potential causes:

* out-of-memory
* unexpected exceptions
* process termination
* deployment
* machine failure
* dependency failure

The queue system should allow unfinished jobs to become available again according to its delivery semantics.

---

# Graceful Shutdown

Workers should shut down gracefully.

When the application receives a termination signal:

```text
SIGTERM
 ↓
Stop accepting new jobs
 ↓
Finish currently processing safe jobs
 ↓
Close resources
 ↓
Exit
```

Do not abruptly terminate workers while they are halfway through important operations unless unavoidable.

---

# Deployment Considerations

Deployments can interrupt workers.

Plan for:

```text
old worker
 ↓
shutdown
 ↓
new worker
```

The queue should remain durable across worker restarts.

Avoid relying entirely on worker memory for job state.

---

# Queue Durability

Important jobs should not disappear because a worker or server restarted.

For persistent work, use durable queue storage.

Do not treat:

```js
const jobs = [];
```

as a production queue.

An in-memory queue can be useful for prototypes or non-critical temporary work, but it is not equivalent to durable background processing.

---

# Queue Storage

Depending on the project, queues may be backed by:

* Redis
* dedicated message brokers
* cloud queue services
* database-backed queues

Choose infrastructure according to:

* workload
* reliability requirements
* deployment environment
* existing architecture
* operational complexity
* expected scale

Do not add infrastructure unnecessarily.

---

# Queue Technology Selection

Before choosing a queue system, consider:

```text
Does the project already use Redis?
Does the deployment support persistent workers?
Is job durability required?
How many jobs are expected?
Are delayed jobs required?
Are priorities required?
Are retries required?
Is scheduling required?
Is monitoring available?
```

Use the technology that fits the project rather than automatically choosing the most sophisticated queue system.

---

# Redis-Based Queues

Redis-backed queues can be useful for:

* background jobs
* delayed jobs
* retries
* worker coordination
* temporary job state

If Redis is already part of the architecture, it may be a practical choice.

Do not add Redis solely because it is popular if the project does not need a queue.

---

# Database-Backed Jobs

For smaller systems, database-backed job processing may sometimes be sufficient.

For example:

```text
jobs collection
 ↓
worker polls pending jobs
 ↓
process job
 ↓
update status
```

This can reduce infrastructure complexity.

However, database polling must be implemented carefully to avoid:

* duplicate jobs
* excessive queries
* race conditions
* database contention

---

# Job Claiming

When multiple workers use the same job store, workers must safely claim jobs.

Avoid:

```text
Worker A → sees job
Worker B → sees same job
Worker A → processes
Worker B → processes
```

Prefer an atomic claim mechanism:

```text
Worker A → atomically claims job
Worker B → cannot claim same job
```

Use database or queue features designed for safe job claiming.

---

# Visibility Timeout

Some queue systems temporarily hide a job while a worker processes it.

Example:

```text
Job
 ↓
Worker claims
 ↓
Job hidden
 ↓
Worker crashes
 ↓
Visibility expires
 ↓
Job becomes available again
```

This prevents permanently lost jobs while avoiding immediate duplicate processing.

The timeout must be appropriate for expected job duration.

---

# Heartbeats

For very long jobs, workers may need to periodically signal that they are still processing the job.

Example:

```text
Worker
 ↓
heartbeat
 ↓
heartbeat
 ↓
heartbeat
 ↓
complete
```

This can prevent a long-running job from being incorrectly considered abandoned.

Use heartbeats only when the queue system requires or benefits from them.

---

# Job Progress

For long-running jobs, users may need progress information.

Example:

```json
{
  "status": "PROCESSING",
  "progress": 65
}
```

Possible progress states:

```text
0%
25%
50%
75%
100%
```

Progress should reflect meaningful work.

Do not fake progress merely to make the UI look active.

---

# Progress Tracking

A worker can update:

```text
job status
progress
current stage
processed count
total count
error information
```

For example:

```text
PROCESSING
Stage: Generating spreadsheet
Progress: 70%
Processed: 700 / 1000
```

---

# Job Status API

For asynchronous user-facing operations, provide a way to retrieve job state when needed.

Example:

```text
POST /reports
     ↓
jobId
```

Then:

```text
GET /jobs/:jobId
```

Possible response:

```json
{
  "status": "PROCESSING",
  "progress": 70
}
```

Do not expose internal queue details unnecessarily.

---

# Notifications

Polling is not the only way to communicate job completion.

Depending on the application, consider:

```text
polling
WebSocket
Server-Sent Events
push notification
email
```

Use the simplest method that satisfies the UX requirement.

---

# Polling Job Status

For a simple application, the frontend may periodically request:

```text
GET /jobs/:id
```

Example:

```text
2 seconds
4 seconds
8 seconds
```

Do not poll excessively.

If the job completes, stop polling.

---

# Event-Driven Completion

For applications requiring real-time updates:

```text
Worker
 ↓
Job completed
 ↓
Event
 ↓
WebSocket / SSE
 ↓
Client
```

This avoids unnecessary repeated status requests.

Use real-time communication when the UX actually benefits from it.

---

# Job Results

Do not store huge results directly inside job metadata unless the queue system is designed for it.

Prefer:

```text
Worker
 ↓
Generate result
 ↓
Store result
 ↓
Job stores reference
```

For example:

```json
{
  "status": "COMPLETED",
  "resultUrl": "...",
  "resultId": "..."
}
```

---

# Temporary Results

Large generated files should usually live in appropriate storage rather than remaining inside the queue.

Example:

```text
Worker
 ↓
Generate PDF
 ↓
Cloud/object storage
 ↓
Store URL/reference
 ↓
Mark job completed
```

---

# Job Ownership

If jobs are user-specific, verify that a user can only access their own jobs.

For example:

```text
GET /jobs/:jobId
```

must not allow:

```text
User A
 ↓
request User B's job
 ↓
receive sensitive information
```

Job status endpoints require normal authentication and authorization considerations.

---

# Job Security

Never trust queue payloads blindly.

Validate:

* job type
* identifiers
* permissions
* referenced records
* expected data shape

A job placed into the queue should not bypass normal security assumptions.

---

# Sensitive Data

Avoid placing unnecessary sensitive information directly into queue payloads.

Prefer references:

```text
userId
recordId
documentId
```

rather than copying:

```text
password
authentication tokens
full private documents
unnecessary personal information
```

---

# External API Jobs

Queues are particularly useful for external integrations.

Example:

```text
API
 ↓
Queue
 ↓
Worker
 ↓
External API
```

Benefits include:

* controlled request rate
* retries
* backoff
* failure isolation
* asynchronous processing

---

# Rate-Limited External Services

If an external API allows only:

```text
100 requests / minute
```

do not allow:

```text
50 workers
×
10 concurrent requests
```

to overwhelm it.

Implement appropriate:

* concurrency limits
* rate limits
* backoff
* scheduling

---

# Webhook Processing

Webhook handlers should generally acknowledge quickly when possible.

Instead of:

```text
Webhook
 ↓
Perform complex processing
 ↓
Call external services
 ↓
Database operations
 ↓
Respond
```

consider:

```text
Webhook
 ↓
Validate
 ↓
Persist/queue event
 ↓
Respond quickly
```

Then:

```text
Queue
 ↓
Worker
 ↓
Process webhook
```

This reduces webhook timeout risk.

---

# Duplicate Webhooks

External systems may send the same webhook more than once.

Use the provider's event ID or another reliable idempotency identifier when available.

Example:

```text
eventId
 ↓
already processed?
 ├── yes → ignore safely
 └── no → process
```

---

# Ordering

Some workloads require jobs to be processed in order.

For example:

```text
Update A
Update B
Update C
```

may need to remain:

```text
A → B → C
```

Do not assume queues automatically preserve the business-level ordering your application needs.

---

# Ordered Jobs

If ordering matters, design explicitly using mechanisms such as:

* per-entity queues
* partitioning
* sequence numbers
* locks
* ordered consumers

Avoid forcing global ordering across the entire system unless absolutely necessary.

Global ordering can reduce throughput.

---

# Per-Entity Ordering

Often only related jobs need ordering.

For example:

```text
Driver A
 → update 1
 → update 2
 → update 3

Driver B
 → update 1
 → update 2
```

Driver A's jobs may need ordering while Driver B's jobs can execute concurrently.

This allows better throughput.

---

# Race Conditions

Queues do not automatically eliminate race conditions.

Two jobs may still update the same record.

Example:

```text
Job A → read balance = 100
Job B → read balance = 100

Job A → write 90
Job B → write 80
```

The final result may be incorrect.

Use appropriate:

* atomic database operations
* transactions
* optimistic concurrency
* locks
* version checks

when required.

---

# Optimistic Concurrency

A worker can verify that the record has not changed before applying its update.

Example concept:

```text
Read version = 5
 ↓
Process
 ↓
Update only if version = 5
```

If another operation changed it:

```text
version = 6
```

the worker can detect the conflict.

---

# Distributed Locks

Locks may be appropriate when multiple workers must not perform the same operation simultaneously.

Use locks carefully.

A poorly designed lock can create:

* deadlocks
* stale locks
* bottlenecks
* unavailable resources

Prefer database or queue mechanisms designed for the problem before implementing custom distributed locking.

---

# Lock Expiration

If a lock is used, consider what happens when the worker crashes.

Avoid:

```text
Worker acquires lock
 ↓
Worker crashes
 ↓
Lock remains forever
```

Use expiration or ownership mechanisms where appropriate.

---

# Batch Processing

Queues are useful for large workloads.

Instead of:

```text
Process 100,000 records
```

inside one huge job, consider:

```text
Job
 ↓
Batch 1
Batch 2
Batch 3
...
Batch 100
```

This can improve:

* memory usage
* retry granularity
* progress tracking
* recovery
* worker throughput

---

# Batch Size

Avoid extremely small batches:

```text
1 record per job
```

if job overhead becomes significant.

Avoid extremely large batches:

```text
100,000 records per job
```

if failure recovery becomes difficult.

Choose a batch size based on:

* processing cost
* memory usage
* database limits
* retry requirements
* expected job duration

---

# Partial Failure

Large jobs can partially succeed.

Example:

```text
1000 records
 ↓
700 succeeded
 ↓
300 failed
```

Define what should happen.

Possible strategies:

```text
retry entire job
retry failed records
mark job partially completed
send failed records to dead-letter processing
```

Do not automatically retry everything if most of the work already succeeded.

---

# Transaction Boundaries

Do not assume an entire large background job must be one database transaction.

Large transactions can create contention.

Prefer smaller atomic units when appropriate.

---

# Job Cancellation

Some jobs should be cancellable.

Example:

```text
User starts large report
 ↓
User cancels
```

The worker should be able to recognize cancellation when practical.

Possible lifecycle:

```text
QUEUED
 ↓
PROCESSING
 ↓
CANCEL_REQUESTED
 ↓
CANCELLED
```

Cancellation should not leave partially updated data in an invalid state.

---

# Cooperative Cancellation

Workers can periodically check:

```text
Has the job been cancelled?
```

For example:

```text
process batch
 ↓
check cancellation
 ↓
process next batch
 ↓
check cancellation
```

This is particularly useful for long-running batch jobs.

---

# Queue Cleanup

Completed jobs should not remain forever unless required.

Define retention policies for:

* completed jobs
* failed jobs
* dead-letter jobs
* job results
* logs

Keep enough history for debugging and auditing without creating unlimited storage growth.

---

# Job Retention

For example:

```text
Completed jobs
→ short retention

Failed jobs
→ longer retention

Dead-letter jobs
→ longer retention for investigation
```

The exact retention period depends on the project's requirements.

---

# Observability

Queue systems must be observable.

Track:

* queue depth
* processing rate
* waiting time
* processing time
* success rate
* failure rate
* retry count
* dead-letter count
* active workers
* worker utilization
* job age
* timeout count

Without these metrics, queue problems can become difficult to diagnose.

---

# Queue Metrics

Useful measurements include:

```text
jobs added / minute
jobs completed / minute
jobs failed / minute
average processing time
p95 processing time
queue wait time
oldest job age
retry rate
dead-letter rate
```

Do not rely only on average processing time.

Tail latency matters.

---

# Worker Logs

Worker logs should identify the job without exposing sensitive information.

Useful information:

```text
job ID
job type
worker ID
attempt number
start time
duration
result
error category
```

Avoid logging:

```text
passwords
tokens
private documents
sensitive personal information
```

---

# Correlation IDs

When a request creates a background job, preserve a correlation identifier where appropriate.

Example:

```text
HTTP Request
correlationId = abc123
      ↓
Queue Job
correlationId = abc123
      ↓
Worker
correlationId = abc123
```

This helps trace:

```text
request
→ job
→ worker
→ database
→ external service
```

---

# Monitoring Queue Health

A healthy queue generally has:

```text
processing rate >= incoming rate
```

over sustained periods.

Watch for:

```text
queue continuously growing
oldest job becoming older
workers repeatedly crashing
high retry rates
high processing latency
```

---

# Alerting

Consider alerts for:

* queue depth above threshold
* oldest job too old
* worker failure rate
* dead-letter jobs
* repeated job failures
* excessive retry rates
* worker count unexpectedly low
* external dependency failures

Alerts should indicate actionable problems.

Do not create alerts for every minor fluctuation.

---

# Queue Performance

Measure both:

```text
queue waiting time
```

and:

```text
job processing time
```

Example:

```text
Queue wait = 30 seconds
Processing = 2 seconds
```

The worker is fast.

The problem is insufficient worker capacity.

Another example:

```text
Queue wait = 1 second
Processing = 30 seconds
```

The worker workload itself is slow.

These require different solutions.

---

# Scaling Workers

Workers can often be scaled independently.

Example:

```text
API:
2 instances

Workers:
8 instances
```

This is useful when background processing requires significantly more resources than normal API traffic.

---

# Autoscaling Workers

For larger systems, worker count can respond to queue demand.

Example:

```text
Queue small
 ↓
2 workers

Queue growing
 ↓
5 workers

Queue very large
 ↓
10 workers
```

Only introduce autoscaling when workload and infrastructure justify it.

---

# Do Not Over-Scale

More workers can increase contention.

For example:

```text
100 workers
 ↓
100 simultaneous database operations
 ↓
database overloaded
 ↓
everything becomes slower
```

Scaling should consider the entire system.

---

# Worker Resource Isolation

If jobs are CPU-heavy, consider separate worker resources.

If jobs are memory-heavy, isolate them.

If jobs call an external API, control their request rate.

Different workloads may need different worker configurations.

---

# Failure Isolation

A failure in one worker should not bring down the entire backend.

Prefer:

```text
API
 ├── Worker A
 ├── Worker B
 ├── Worker C
```

rather than tightly coupling all background work to the API process.

---

# Worker Health

Workers should expose or report useful health information.

Monitor:

```text
alive
processing
idle
stuck
restarting
```

Do not consider a process "healthy" simply because it has not crashed.

A worker that is alive but has processed nothing for an hour may still be unhealthy.

---

# Stuck Jobs

Detect jobs that remain in:

```text
PROCESSING
```

for unexpectedly long periods.

Possible causes:

* dead worker
* external API hanging
* database lock
* infinite loop
* resource exhaustion

Use timeouts and visibility mechanisms where appropriate.

---

# Graceful Recovery

A robust system should recover from:

```text
worker crash
server restart
deployment
database interruption
external API outage
network failure
```

without silently losing important work.

---

# Backend Architecture Example

For a typical Express application:

```text
src/
├── controllers/
├── routes/
├── services/
├── models/
├── middleware/
├── queues/
│   ├── queueConfig.js
│   ├── reportQueue.js
│   └── notificationQueue.js
├── workers/
│   ├── reportWorker.js
│   └── notificationWorker.js
├── jobs/
│   ├── reportJob.js
│   └── notificationJob.js
└── app.js
```

This is an example.

Do not create all of these folders if the project only needs one simple background worker.

---

# Queue Responsibility

Queue modules should primarily define:

* queue configuration
* job creation
* job options
* retry behavior
* scheduling
* priority

They should not contain large business logic implementations.

---

# Worker Responsibility

Workers should primarily:

* receive jobs
* validate job data
* call the appropriate service
* update job state
* handle errors
* report progress where necessary

Avoid putting the entire business logic directly inside the queue configuration.

---

# Service Responsibility

Business logic should generally remain in services.

For example:

```text
Worker
 ↓
reportService.generateReport()
```

rather than:

```text
Worker
 ↓
500 lines of report-generation logic
```

This keeps the business logic reusable and testable.

---

# Example Architecture

```text
POST /reports
      ↓
Report Controller
      ↓
Report Service
      ↓
Create Job
      ↓
Report Queue
      ↓
Report Worker
      ↓
Report Service
      ↓
Database
      ↓
File Storage
      ↓
Job COMPLETED
```

This keeps request handling and background processing separated while reusing business logic.

---

# Testing Queues

Test:

* job creation
* worker processing
* successful completion
* invalid jobs
* retries
* backoff
* duplicate execution
* dead-letter handling
* worker crashes
* job timeouts
* cancellation
* partial failures
* large workloads

Do not test only the happy path.

---

# Testing Idempotency

Run the same job multiple times:

```text
Job A
Job A
Job A
```

and verify that the final state remains correct.

This is especially important for:

* payments
* notifications
* database updates
* external API calls
* webhook processing

---

# Testing Failure Recovery

Simulate:

```text
worker crashes
database unavailable
external API timeout
network failure
invalid job
```

Then verify that the system:

* retries when appropriate
* does not retry permanent failures forever
* does not lose important work
* does not create duplicate side effects
* eventually exposes failures

---

# Testing Queue Load

For large workloads, test:

```text
10 jobs
100 jobs
1,000 jobs
10,000 jobs
```

where appropriate.

Measure:

* throughput
* queue wait time
* memory
* CPU
* database load
* worker utilization

---

# Queue Anti-Patterns

Avoid:

* using queues for simple CRUD operations
* keeping long-running HTTP requests open unnecessarily
* storing huge payloads in jobs
* trusting job data without validation
* assuming exactly-once execution
* processing jobs without idempotency
* unlimited worker concurrency
* unlimited retries
* immediate retries
* retrying permanent failures
* allowing poison jobs to retry forever
* ignoring dead-letter jobs
* storing critical job state only in memory
* using `setTimeout()` for important persistent scheduled jobs
* allowing queues to grow without monitoring
* creating one giant job for an enormous dataset
* running CPU-heavy work directly on the main event loop
* allowing one workload to consume all workers
* ignoring database contention
* ignoring external API rate limits
* creating unnecessary queue infrastructure
* putting business logic directly into queue configuration
* hiding failed jobs instead of making them observable
* claiming a job completed when only the job was accepted

---

# Queue Implementation Decision Tree

When adding a new backend operation, ask:

```text
Does the user need the result immediately?
        │
       Yes
        │
        ▼
Can it complete quickly?
        │
       Yes
        │
        ▼
Handle synchronously
```

If:

```text
Does the user need the result immediately?
        │
        No
        │
        ▼
Is the work expensive, long-running,
bursty, retryable, or independently scalable?
        │
       Yes
        │
        ▼
Use a queue/worker
```

If:

```text
No
 ↓
Keep the implementation simple
```

---

# Production Readiness Checklist

Before using queues for important production workloads:

* [ ] Jobs are durable where required.
* [ ] Job payloads are small.
* [ ] Job types are explicit.
* [ ] Job states are defined.
* [ ] Workers validate job data.
* [ ] Workers are idempotent where necessary.
* [ ] Retryable failures are identified.
* [ ] Retry limits exist.
* [ ] Exponential backoff is configured where appropriate.
* [ ] Jitter is used where appropriate.
* [ ] Poison jobs cannot retry forever.
* [ ] Dead-letter handling exists where necessary.
* [ ] Job timeouts exist where appropriate.
* [ ] Worker shutdown is graceful.
* [ ] Worker crashes can be recovered from.
* [ ] Concurrency is controlled.
* [ ] External API limits are respected.
* [ ] Database contention is considered.
* [ ] Queue depth is observable.
* [ ] Queue age is observable.
* [ ] Worker health is observable.
* [ ] Processing time is measured.
* [ ] Failed jobs are visible.
* [ ] Sensitive data is not unnecessarily placed in jobs.
* [ ] Job ownership and authorization are enforced.
* [ ] Large workloads are batched when appropriate.
* [ ] Job results are stored appropriately.
* [ ] Old jobs and results have retention policies.
* [ ] Load testing has been performed when required.

---

# Final Principle

A queue is not a replacement for good backend architecture.

It is a tool for controlling **when, where, and how background work happens**.

Prefer:

**short HTTP requests over long blocking requests**

**durable jobs over in-memory background tasks**

**controlled concurrency over unlimited parallelism**

**idempotent workers over duplicate side effects**

**backoff over immediate retries**

**dead-letter handling over infinite retries**

**small job payloads over huge queue messages**

**batch processing over massive single jobs**

**observable workers over invisible background processes**

**independent worker scaling over forcing API servers to perform everything**

**simple architecture over unnecessary infrastructure**

The final system should make an intentional distinction between:

```text
Work that must happen now
```

and:

```text
Work that can happen reliably in the background.
```

The goal is not to put everything into a queue.

The goal is to make expensive, asynchronous, retryable, or bursty work **safe, controlled, observable, recoverable, and independent from user-facing request performance.**

```
```
