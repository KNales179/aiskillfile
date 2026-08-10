````md
---
name: backend-observability
description: Build observable backend systems with structured logging, request IDs, error tracking, metrics, health checks, performance monitoring, tracing, audit logging, database and external-service visibility, and actionable diagnostics. Use this skill whenever developing, reviewing, debugging, or deploying backend systems where understanding requests, failures, performance, system health, background jobs, database behavior, or security-relevant activity is important. Implement observability proportionally to the project's size and requirements rather than adding unnecessary monitoring complexity.
---

# Backend Observability

Build backend systems that are easy to understand when they are running correctly and easy to diagnose when something goes wrong.

Observability should allow developers and administrators to answer questions such as:

- What happened?
- When did it happen?
- Which request caused it?
- Which user or system initiated it?
- Which endpoint was involved?
- How long did it take?
- Did the database cause the delay?
- Did an external service fail?
- Was the failure caused by invalid input?
- Did the request reach the intended controller?
- Did a background job fail?
- Is the server healthy?
- Is the database healthy?
- Is the system becoming slower?
- Are errors increasing?
- Is a specific endpoint repeatedly failing?

The goal is not to log everything.

The goal is to collect the **right information to understand the system without creating unnecessary noise, performance overhead, storage costs, or security risks.**

Prioritize:

1. Actionable information
2. Structured logs
3. Request traceability
4. Error visibility
5. Performance visibility
6. System health
7. Security awareness
8. Background-job visibility
9. Database visibility
10. External-service visibility
11. Low operational noise
12. Protection of sensitive information

---

# Core Observability Principles

## Observability is not the same as logging

Logging is only one part of observability.

A complete observability strategy can include:

```text
Logs
Metrics
Traces
Health checks
Error tracking
Audit events
Performance measurements
Database monitoring
External dependency monitoring
Background-job monitoring
````

Do not assume:

```text
more logs = more observability
```

A backend producing thousands of lines of unstructured logs can be harder to diagnose than a backend producing a smaller amount of well-structured information.

---

# The Three Core Signals

Use the three traditional observability signals when the project's complexity justifies them.

## Logs

Logs explain individual events.

Examples:

```text
Request received
Authentication failed
Driver created
Database query failed
External service timed out
Background job completed
```

---

## Metrics

Metrics show system behavior over time.

Examples:

```text
requests per minute
error rate
average latency
95th percentile latency
database connection usage
queue depth
job processing time
memory usage
CPU usage
```

Metrics are especially useful for detecting trends.

---

## Traces

Traces show how one request travels through the system.

Example:

```text
HTTP Request
   ↓
Authentication
   ↓
Controller
   ↓
Service
   ↓
MongoDB
   ↓
Cloudinary
   ↓
Response
```

Tracing becomes increasingly valuable when a backend has multiple services or external dependencies.

Do not introduce distributed tracing into a tiny backend unless it provides real value.

---

# Observability by Project Size

## Small Project

A small application may only need:

```text
structured logging
request IDs
error handling
health endpoint
basic performance timing
database connection monitoring
```

This is often enough.

---

## Medium Project

Consider adding:

```text
metrics
error tracking
audit logs
background-job monitoring
external dependency monitoring
```

---

## Large Project

Consider:

```text
distributed tracing
centralized logs
metrics dashboards
alerting
service-level indicators
queue monitoring
database performance monitoring
dependency health monitoring
automated incident detection
```

Do not automatically implement the large-project architecture.

Scale observability with the system.

---

# Structured Logging

Prefer structured logs over arbitrary strings.

Avoid:

```js
console.log("something happened with driver");
```

Prefer a structured event such as:

```json
{
  "level": "info",
  "event": "driver.updated",
  "driverId": "123",
  "adminId": "456"
}
```

Structured logs are easier to:

* search
* filter
* aggregate
* analyze
* alert on
* connect with other logs

---

# Log Levels

Use log levels intentionally.

Common levels include:

```text
debug
info
warn
error
fatal
```

The exact set depends on the logging library.

---

# Debug

Use `debug` for detailed information useful during development or diagnosis.

Examples:

```text
query parameters
internal state transitions
cache decisions
development diagnostics
```

Do not log sensitive information merely because it is useful for debugging.

---

# Info

Use `info` for normal meaningful system events.

Examples:

```text
server started
database connected
request completed
driver created
background job completed
```

Do not log every tiny internal operation at info level.

---

# Warn

Use `warn` for unusual conditions that do not necessarily indicate failure.

Examples:

```text
retry triggered
slow request detected
deprecated API used
external service temporarily unavailable
unusually large request
```

A warning should mean something worth investigating if it becomes frequent.

---

# Error

Use `error` when an operation failed and requires attention or diagnosis.

Examples:

```text
database operation failed
external API failed
unexpected controller error
background job failed
file processing failed
```

Include enough context to understand what failed.

---

# Fatal

Use `fatal` only for severe failures that prevent the application from functioning.

Examples may include:

```text
critical startup dependency unavailable
irrecoverable initialization failure
```

Do not use fatal simply because an individual request failed.

---

# Log Events

Prefer meaningful event names.

Examples:

```text
auth.login.success
auth.login.failed
driver.created
driver.updated
driver.deleted
transaction.approved
transaction.rejected
upload.completed
upload.failed
job.started
job.completed
job.failed
database.connected
database.disconnected
external_service.timeout
```

This makes logs easier to search and aggregate.

---

# Request IDs

Every API request should ideally have a unique request identifier.

Example:

```text
X-Request-ID: 8f31c7...
```

The identifier should follow the request through relevant backend operations.

For example:

```text
Request
  ↓
Controller
  ↓
Service
  ↓
Database
  ↓
External API
```

All relevant logs should be associated with the same request ID.

---

# Why Request IDs Matter

Without request IDs:

```text
Request A
Request B
Request C
Request A error
Request C database query
Request B response
```

can become difficult to correlate.

With request IDs:

```text
requestId=abc
requestId=abc
requestId=abc
requestId=abc
```

the entire operation becomes traceable.

---

# Request ID Generation

If the client provides a request ID, do not blindly trust arbitrary values.

Depending on the system, either:

* validate it
* generate a server-side identifier
* accept a validated client identifier

The identifier should be safe to log and suitable for correlation.

---

# Do Not Put Sensitive Information in Request IDs

Request IDs should never contain:

```text
password
email
phone number
token
JWT
API key
personal document number
```

Use random identifiers instead.

---

# Request Logging

A useful request log may contain:

```text
requestId
method
route
status
duration
timestamp
userId when appropriate
```

Example:

```json
{
  "level": "info",
  "event": "http.request.completed",
  "requestId": "abc123",
  "method": "GET",
  "route": "/api/v1/drivers",
  "statusCode": 200,
  "durationMs": 84
}
```

---

# Do Not Log Every Request Blindly

Logging every request can become noisy.

Determine whether the project actually benefits from:

```text
every request
only errors
slow requests
important administrative actions
```

Production systems often need request visibility, but the amount of detail should be controlled.

---

# Request Duration

Measure request duration.

Example:

```text
GET /drivers
duration = 82ms
```

This allows detection of slow endpoints.

Track useful latency statistics such as:

```text
average
median
p95
p99
```

Average latency alone can hide severe slow-request problems.

---

# Slow Request Detection

Consider identifying requests above an intentional threshold.

Example:

```text
normal request
< 500ms

slow request
>= 500ms
```

The threshold should depend on the application.

A slow administrative export may naturally take longer than a simple authentication request.

Do not use one arbitrary threshold for every endpoint.

---

# Error Tracking

Unexpected errors should be captured with enough context to diagnose them.

Useful information may include:

```text
error type
error message
stack trace
request ID
route
method
status code
environment
timestamp
authenticated user ID when appropriate
```

Do not include:

```text
password
access token
refresh token
secret
private key
```

---

# Stack Traces

Keep stack traces in internal error logs.

Do not expose internal stack traces to API clients.

Bad:

```json
{
  "error": "MongoError at /src/controllers/driverController.js:82..."
}
```

Prefer:

```json
{
  "success": false,
  "message": "An unexpected error occurred",
  "error": {
    "code": "INTERNAL_ERROR"
  }
}
```

while keeping the technical details in internal observability systems.

---

# Error Classification

Separate errors into meaningful categories.

For example:

```text
Validation Error
Authentication Error
Authorization Error
Not Found
Conflict
Database Error
External Service Error
Timeout
Rate Limit
Unexpected Internal Error
```

This helps determine whether an error requires:

```text
frontend correction
user action
developer investigation
infrastructure intervention
security investigation
```

---

# Expected vs Unexpected Errors

Not every error should be treated as an incident.

For example:

```text
wrong password
```

is an expected authentication failure.

Whereas:

```text
database connection unexpectedly terminated
```

may require investigation.

Do not fill error dashboards with expected user mistakes.

---

# Authentication Logging

Security-relevant authentication events should be observable.

Examples:

```text
login.success
login.failed
logout
token.refresh.failed
password.changed
password.reset.requested
email.verification.completed
account.locked
2fa.failed
trusted_device.added
trusted_device.removed
```

Do not log passwords or authentication tokens.

---

# Authorization Logging

For important authorization failures, record enough context to detect suspicious behavior.

Example:

```json
{
  "level": "warn",
  "event": "authorization.denied",
  "requestId": "abc123",
  "userId": "user123",
  "resource": "transactions",
  "action": "approve"
}
```

Do not expose sensitive information unnecessarily.

---

# Security Event Monitoring

Certain patterns may deserve special attention:

```text
many failed logins
many authorization failures
repeated invalid tokens
unexpected privilege changes
repeated password reset attempts
suspicious request frequency
```

The goal is not to log every security event as an emergency.

The goal is to make suspicious behavior detectable.

---

# Audit Logs

Audit logs answer:

```text
Who did what?
When?
To which resource?
What changed?
```

For administrative systems, important actions may include:

```text
created record
updated record
deleted record
changed status
approved transaction
rejected transaction
uploaded document
changed permissions
changed account details
```

---

# Audit Log vs Application Log

Do not treat them as identical.

Application log:

```text
driver.updated
```

Audit event:

```text
Admin 123 changed Driver 456
status:
REGISTERED → INACTIVE
```

Application logs help developers understand system behavior.

Audit logs provide a historical record of meaningful user actions.

---

# Audit Log Integrity

For important systems, audit records should not be casually editable or removable by ordinary users.

Consider:

```text
append-only behavior
restricted access
retention rules
timestamps
actor identity
resource identity
action
before/after values where appropriate
```

The level of protection should match the project's requirements.

---

# Database Observability

Monitor important database events.

Examples:

```text
connection established
connection lost
query failure
transaction failure
slow query
connection pool exhaustion
```

For MongoDB or another database, observe the relevant database-specific health signals.

---

# Database Connection Health

The backend should be able to determine whether its database dependency is healthy.

For example:

```text
API server
    ↓
MongoDB
    ↓
connected
```

If the database becomes unavailable:

```text
API server
    ↓
MongoDB
    X
connection failed
```

the system should produce a useful diagnostic signal.

---

# Do Not Spam Database Logs

Avoid logging every successful query unless debugging requires it.

Query-level logging can produce enormous noise and performance overhead.

Prefer:

```text
slow queries
failed queries
important operations
aggregated metrics
```

when possible.

---

# Slow Database Operations

Monitor queries that exceed an appropriate duration.

For example:

```text
query duration = 1200ms
```

may warrant investigation.

A useful database performance log can include:

```text
operation
collection
duration
requestId
query category
```

Do not log sensitive query parameters unnecessarily.

---

# Database Errors

When a database error occurs, log enough context to identify:

```text
which operation failed
which collection/model was involved
which request caused it
whether it was retryable
```

Do not expose raw database error details to clients.

---

# External Services

External dependencies should be observable.

Examples:

```text
Cloudinary
email provider
payment gateway
maps API
geocoding service
third-party authentication
```

Track:

```text
request count
success count
failure count
latency
timeouts
retries
```

---

# External Service Timeouts

When an external service times out, record:

```text
service
operation
duration
request ID
retry attempt
final result
```

Example:

```json
{
  "level": "warn",
  "event": "external_service.timeout",
  "service": "email",
  "operation": "sendVerificationEmail",
  "durationMs": 5000,
  "attempt": 2,
  "requestId": "abc123"
}
```

---

# External Dependency Failures

Do not allow external service failures to become invisible.

If:

```text
Cloudinary fails
```

the system should be able to show that the upload failed because of the dependency rather than merely:

```text
500 Internal Server Error
```

internally.

The user-facing message can remain simple.

The internal diagnostic should be specific.

---

# Health Checks

Provide a health endpoint when useful.

A basic example:

```text
GET /health
```

could return:

```json
{
  "success": true,
  "status": "healthy"
}
```

Do not make a health endpoint unnecessarily expensive.

---

# Liveness vs Readiness

For larger deployments, distinguish:

```text
Liveness
```

from:

```text
Readiness
```

Liveness asks:

```text
Is the process alive?
```

Readiness asks:

```text
Can this instance actually serve traffic?
```

A service can be alive while its database connection is unavailable.

---

# Health Check Depth

Avoid making the basic health check depend on every external service.

For example:

```text
/health
```

should not necessarily fail because:

```text
Cloudinary is temporarily unavailable
```

when the API can still serve most requests.

Consider separate dependency checks when needed.

---

# Readiness Checks

A readiness check may verify critical dependencies such as:

```text
database
required configuration
critical internal dependencies
```

Do not include optional services if their failure should not prevent the application from serving unrelated traffic.

---

# Health Response Design

Keep health responses safe.

Do not expose:

```text
database credentials
connection strings
API keys
internal hostnames
stack traces
```

to public clients.

---

# System Metrics

Track basic system metrics when appropriate:

```text
CPU
memory
event-loop delay
process uptime
request count
error count
latency
```

For Node.js systems, event-loop health can be especially useful because a blocked event loop can make the entire server appear slow.

---

# Event Loop Monitoring

A Node.js backend can have:

```text
CPU usage = normal
database = healthy
network = healthy
```

and still be slow because the event loop is blocked.

Watch for expensive synchronous operations such as:

```text
large synchronous computation
large synchronous file operations
CPU-heavy processing
poorly implemented loops
```

Do not use synchronous operations for expensive work on request paths unless there is a strong reason.

---

# Memory Monitoring

Monitor memory when the application is long-running or handles large data.

Look for:

```text
steady memory growth
unexpected spikes
frequent garbage collection pressure
out-of-memory crashes
```

A single memory measurement is less useful than observing the trend.

---

# CPU Monitoring

High CPU can indicate:

```text
CPU-heavy work
large transformations
poor algorithms
unexpected traffic
infinite loops
resource abuse
```

Correlate CPU increases with request volume and endpoint behavior.

---

# Queue Observability

When using background queues, observe:

```text
queue depth
jobs waiting
jobs processing
jobs completed
jobs failed
job duration
retry count
oldest job age
```

Example:

```text
Queue: report-generation

waiting: 12
processing: 2
failed: 1
oldestJobAge: 48s
```

A growing queue can indicate that workers cannot keep up with incoming work.

---

# Worker Observability

Workers should produce useful lifecycle events:

```text
job.started
job.completed
job.failed
job.retrying
worker.started
worker.stopped
worker.error
```

Include:

```text
job ID
job type
attempt
duration
request ID when available
```

Do not log entire job payloads if they contain sensitive information.

---

# Job Correlation

If an API request creates a background job:

```text
POST /exports
```

the request ID should be associated with the job where practical.

Example:

```text
HTTP request
requestId = abc123
       ↓
jobId = export789
       ↓
worker
       ↓
job.completed
requestId = abc123
jobId = export789
```

This makes asynchronous operations much easier to diagnose.

---

# Retry Observability

When retrying an operation, log:

```text
operation
attempt number
reason
delay
final result
```

Example:

```json
{
  "level": "warn",
  "event": "external_service.retrying",
  "service": "email",
  "attempt": 2,
  "reason": "timeout",
  "delayMs": 1000
}
```

Do not create one error event for every temporary retry if the operation ultimately succeeds unless the project needs that level of visibility.

---

# Circuit Breaker Observability

If the system uses circuit breakers, monitor:

```text
closed
open
half-open
```

and record transitions.

Example:

```text
external_service.circuit_opened
external_service.circuit_closed
```

This can explain why requests are failing quickly instead of waiting for a dependency timeout.

---

# API Metrics

Useful API metrics include:

```text
total requests
successful requests
failed requests
requests by endpoint
requests by status code
latency
requests by method
```

Consider grouping dynamic routes properly.

Avoid creating a unique metric for every:

```text
/users/123
/users/124
/users/125
```

Use a route template:

```text
/users/:id
```

instead.

---

# Error Rate

Monitor the proportion of failed requests.

For example:

```text
error rate =
failed requests / total requests
```

A sudden increase can indicate:

```text
deployment problem
database outage
external service outage
bug
traffic spike
security attack
```

---

# Latency Percentiles

Do not rely exclusively on averages.

For example:

```text
average = 100ms
p95 = 800ms
p99 = 3000ms
```

This means most requests may be fast while a smaller group is experiencing severe delays.

Percentiles provide a better view of tail latency.

---

# Endpoint-Level Metrics

Measure important endpoints separately.

For example:

```text
GET /drivers
POST /drivers
PATCH /drivers/:id
POST /auth/login
POST /uploads
```

Different endpoints can have dramatically different performance profiles.

---

# Business Metrics

Observability does not have to be limited to technical infrastructure.

Meaningful business metrics can include:

```text
drivers registered
violations recorded
transactions processed
rides completed
failed payments
documents uploaded
accounts verified
```

These can help determine whether the system is actually functioning from the business perspective.

---

# Do Not Confuse Business Metrics With Logs

A log:

```text
transaction.created
```

records an event.

A metric:

```text
transactions_created_total = 18420
```

allows trends and aggregation.

Both can be useful.

---

# Alerting

Alerts should represent conditions that require attention.

Good alerts include:

```text
API error rate exceeds threshold
database unavailable
queue backlog continuously increasing
worker repeatedly failing
critical endpoint latency severely degraded
service is not ready
```

Avoid alerts for every warning.

Too many alerts create:

```text
alert fatigue
```

which eventually causes important alerts to be ignored.

---

# Alert Severity

Not all incidents have equal importance.

Consider levels such as:

```text
Info
Warning
Critical
```

or another project-appropriate severity system.

Example:

```text
temporary external API timeout
→ warning

database unavailable
→ critical
```

---

# Alert Conditions

Prefer alerts based on sustained conditions rather than one isolated event when possible.

For example:

```text
error rate > 10% for 5 minutes
```

may be more useful than:

```text
one request returned 500
```

The exact thresholds should be based on the application's behavior.

---

# Deployment Observability

After deployment, watch:

```text
error rate
latency
database errors
memory
CPU
external dependencies
queue behavior
```

A deployment that technically succeeds can still introduce:

```text
runtime errors
performance regressions
database compatibility problems
unexpected traffic behavior
```

---

# Startup Logging

When the backend starts, log useful non-sensitive information such as:

```text
environment
application version
port
startup time
database connection status
enabled non-sensitive features
```

Do not log:

```text
database URI
JWT secret
API keys
passwords
```

---

# Graceful Shutdown

Observe shutdown events.

For example:

```text
server.shutdown.started
server.shutdown.completed
```

During shutdown:

```text
stop accepting new work
finish safe in-flight work
close workers
close database connections
close external resources
```

The exact behavior depends on the architecture.

---

# Environment Awareness

Different environments may require different logging levels.

For example:

```text
development
→ debug logs

staging
→ detailed operational logs

production
→ useful info/warn/error logs
```

Do not accidentally enable extremely verbose debugging in production.

---

# Log Retention

Logs should not grow indefinitely.

Consider:

```text
retention period
rotation
compression
storage limits
archiving
```

The correct retention period depends on:

* operational requirements
* security requirements
* compliance requirements
* storage cost

---

# Sensitive Data Protection

Never log:

```text
passwords
password hashes
access tokens
refresh tokens
JWTs
API keys
secrets
private keys
credit card data
security answers
verification tokens
```

Be cautious with personal information as well.

Avoid logging entire request bodies by default.

---

# Request Body Logging

Do not automatically log:

```js
console.log(req.body);
```

Request bodies can contain:

```text
password
email
phone
address
documents
tokens
private information
```

If body logging is required for debugging, sanitize it first and disable it in production when possible.

---

# Token Redaction

If headers are logged, redact sensitive headers.

For example:

```text
Authorization: Bearer ********
```

rather than:

```text
Authorization: Bearer eyJhbGciOi...
```

The same applies to:

```text
Cookie
X-API-Key
Authorization
```

and other secret-bearing headers.

---

# Database Credential Protection

Never log:

```text
MONGODB_URI
DATABASE_URL
DB_PASSWORD
```

even during startup diagnostics.

Prefer:

```text
database connected
database: MongoDB
```

without exposing connection credentials.

---

# PII Awareness

Observability data can itself become sensitive.

Before logging user information, ask:

```text
Do I actually need this value?
```

Prefer IDs over full personal information where possible.

For example:

```text
userId=123
```

may be sufficient instead of:

```text
email=john.doe@example.com
phone=...
address=...
```

---

# Correlation Across Systems

When the backend communicates with:

```text
database
queue
worker
Cloudinary
email provider
payment service
```

maintain correlation where possible.

A useful chain is:

```text
requestId
    ↓
operation
    ↓
jobId
    ↓
external request
```

This makes distributed failures easier to diagnose.

---

# Trace Context

For larger systems, use standardized trace context where appropriate.

Tracing can allow:

```text
HTTP request
    ↓
service A
    ↓
service B
    ↓
database
    ↓
external API
```

to be represented as one trace.

Do not manually invent a complex tracing protocol if a suitable standard and tooling already exists.

---

# Distributed Tracing

Distributed tracing becomes particularly useful when:

```text
multiple backend services
multiple workers
external dependencies
complex request chains
```

are involved.

For a simple:

```text
React Native
   ↓
Express
   ↓
MongoDB
```

application, structured logging and request IDs may provide enough visibility.

---

# Health Monitoring vs Observability

Health checks answer:

```text
Is the service healthy right now?
```

Observability answers:

```text
Why is the service behaving this way?
```

Both are useful.

A green health check does not prove that every endpoint is functioning correctly.

---

# Synthetic Monitoring

For important production systems, consider periodically testing critical workflows.

For example:

```text
login
fetch dashboard
create record
read record
```

A synthetic check can detect failures before users report them.

Do not perform destructive operations against production merely for monitoring.

---

# Observability of Critical Workflows

Identify important workflows.

For an administrative system:

```text
login
driver registration
vehicle registration
violation creation
transaction processing
receipt upload
report generation
```

Make failures in these workflows easy to diagnose.

---

# Debugging Workflow

When an issue is reported:

## Step 1

Find the approximate timestamp.

## Step 2

Find the request ID if available.

## Step 3

Search related logs.

## Step 4

Identify:

```text
endpoint
user
operation
status
duration
```

## Step 5

Determine whether the problem came from:

```text
frontend
API
database
external dependency
worker
infrastructure
```

## Step 6

Inspect relevant metrics.

## Step 7

Inspect traces if available.

## Step 8

Identify the root cause rather than merely the visible error.

---

# Example Diagnostic Chain

Suppose a user reports:

```text
"The receipt upload keeps failing."
```

Observability should make it possible to determine:

```text
Request ID
    ↓
POST /violations/:id/receipt
    ↓
Authentication successful
    ↓
Validation successful
    ↓
Cloudinary upload started
    ↓
Cloudinary timeout
    ↓
Retry attempt 1
    ↓
Cloudinary timeout
    ↓
Upload failed
    ↓
API returned 503
```

Without observability:

```text
Upload failed.
```

With observability:

```text
We know exactly where the workflow failed.
```

---

# Development vs Production

Development environments may use:

```text
verbose logs
debug output
stack traces
local request inspection
```

Production should prioritize:

```text
structured logs
useful errors
security
performance
controlled verbosity
```

Never copy development logging behavior into production blindly.

---

# Observability Tool Selection

Choose tools based on actual requirements.

Possible categories include:

```text
logging library
error tracking
metrics system
tracing system
APM
log aggregation
dashboarding
alerting
```

Do not install five monitoring platforms to monitor one Express server.

One well-configured system is better than several overlapping systems.

---

# Node.js / Express Considerations

For an Express backend, consider middleware that captures:

```text
request ID
method
route
status
duration
```

Centralize error handling.

For example:

```text
Request
   ↓
Request ID middleware
   ↓
Authentication
   ↓
Route
   ↓
Controller
   ↓
Error middleware
```

This makes request-level observability consistent.

---

# Central Error Middleware

Unexpected API errors should pass through a consistent error-handling layer.

The error middleware should:

1. Identify the error.
2. Determine the appropriate HTTP status.
3. Log internal diagnostic information.
4. Attach the request ID.
5. Return a safe response.

Do not duplicate the same error logging logic across every controller.

---

# Async Error Handling

Ensure asynchronous controller errors reach the centralized error handler.

Do not allow rejected promises to disappear silently.

A request that fails should produce:

```text
client response
+
internal diagnostic
```

rather than:

```text
client waits
+
server logs nothing
```

---

# Unhandled Errors

Monitor:

```text
uncaught exceptions
unhandled promise rejections
process crashes
```

These are serious signals.

Do not simply hide them with:

```js
process.on(...)
```

and continue running indefinitely if the process is in an unsafe state.

Understand the failure and use an appropriate recovery strategy.

---

# Monitoring Background Tasks

Any background process should have visibility into:

```text
started
completed
failed
retrying
stuck
```

A worker that silently stops processing jobs is an operational failure even if the API server itself remains healthy.

---

# Detecting Stuck Jobs

For queue-based systems, monitor:

```text
oldest job age
job processing duration
worker heartbeat
```

A queue with:

```text
waiting = 0
```

is not necessarily healthy if:

```text
processing = 50
```

and workers are stuck.

---

# Observability for Scheduled Jobs

Scheduled tasks should record:

```text
job name
scheduled time
actual start
duration
result
failure
```

Examples:

```text
daily cleanup
database backup
report generation
data synchronization
```

Do not let scheduled tasks fail silently.

---

# Data Synchronization Monitoring

For systems syncing data between services:

```text
records processed
records succeeded
records failed
last successful sync
last attempted sync
duration
```

This makes stale data detectable.

---

# External API Monitoring

Track:

```text
success rate
failure rate
latency
timeouts
rate limits
retries
```

A third-party dependency can become the bottleneck even when your own application code is functioning correctly.

---

# Rate Limit Visibility

If a dependency returns:

```text
429 Too Many Requests
```

record:

```text
service
endpoint
timestamp
retry behavior
```

Do not blindly retry rate-limit responses indefinitely.

Coordinate with the backend resilience strategy.

---

# Performance Regression Detection

Compare performance over time.

For example:

```text
GET /drivers
last week p95 = 180ms
today p95 = 640ms
```

This is more useful than knowing:

```text
one request took 640ms
```

Track trends for important endpoints.

---

# Deployment Correlation

When performance or error rates suddenly change, correlate them with:

```text
deployment
configuration change
database migration
dependency change
traffic increase
```

Record application version or deployment identifier in observability data.

Example:

```json
{
  "service": "tirs-backend",
  "version": "1.8.2"
}
```

This makes regressions easier to associate with releases.

---

# Feature Flags

If the application uses feature flags, include relevant flag context in diagnostics where useful.

For example:

```text
feature=newUploadFlow
```

This can help identify whether failures are limited to a new feature.

Do not log sensitive feature configuration.

---

# Observability During Development

Do not wait until production to add observability.

At minimum, development should make it easy to see:

```text
request
response
errors
database connection
important operations
```

This reduces debugging time during development.

---

# Testing Observability

Observability itself should be tested.

Verify that:

```text
errors are logged
request IDs are preserved
sensitive values are redacted
health checks work
failed jobs are visible
important audit events are recorded
```

Do not assume monitoring works simply because the logging code exists.

---

# Avoid Logging as a Substitute for Debugging

Bad observability:

```text
console.log("HERE 1")
console.log("HERE 2")
console.log("HERE 3")
console.log("HERE 4")
```

Better:

```json
{
  "event": "driver.update.failed",
  "requestId": "abc123",
  "driverId": "456",
  "reason": "duplicate_license"
}
```

Logs should communicate meaning.

---

# Avoid Duplicate Logs

Do not log the same error at every layer:

```text
controller logs error
service logs error
repository logs error
middleware logs error
```

which creates four identical errors.

Decide which layer owns the log.

Lower layers may add context or rethrow errors without repeatedly logging the same failure.

---

# Avoid Sensitive Debugging

Never temporarily add:

```js
console.log(req.headers);
console.log(req.body);
console.log(process.env);
```

to production debugging.

This can expose:

```text
tokens
passwords
secrets
credentials
personal information
```

Use sanitized diagnostics instead.

---

# Observability Checklist

Before considering backend observability complete:

* [ ] Structured logging exists.
* [ ] Log levels are used intentionally.
* [ ] Request IDs are available.
* [ ] Request duration can be measured.
* [ ] Unexpected errors are centrally captured.
* [ ] Stack traces remain internal.
* [ ] Sensitive data is redacted.
* [ ] Authentication events are observable.
* [ ] Important authorization failures are observable.
* [ ] Important administrative actions have audit records when required.
* [ ] Database connection health is visible.
* [ ] Database failures are observable.
* [ ] Slow database operations can be identified when necessary.
* [ ] External service failures are observable.
* [ ] External service timeouts are visible.
* [ ] Background jobs have lifecycle visibility.
* [ ] Queue backlog can be monitored when queues exist.
* [ ] Worker failures are observable.
* [ ] Health checks exist where appropriate.
* [ ] Readiness is separated from liveness when necessary.
* [ ] API latency can be measured.
* [ ] Error rates can be measured.
* [ ] Important endpoints can be identified individually.
* [ ] CPU and memory can be monitored when necessary.
* [ ] Node.js event-loop health is considered for demanding systems.
* [ ] Deployment versions can be correlated with failures.
* [ ] Alerts exist for genuinely critical conditions.
* [ ] Alerts avoid excessive noise.
* [ ] Logs have appropriate retention.
* [ ] Observability data is access-controlled.
* [ ] Observability itself has been tested.

---

# Anti-Patterns

Avoid:

* logging everything
* logging nothing
* unstructured logs everywhere
* `console.log()` as the entire monitoring strategy
* logging passwords
* logging tokens
* logging secrets
* logging complete request bodies
* exposing stack traces to clients
* exposing database errors to clients
* generating a unique metric for every resource ID
* alerting on every warning
* relying only on average latency
* health checks that are unnecessarily expensive
* health endpoints exposing secrets
* duplicate error logs at every layer
* silently failing background jobs
* workers with no failure visibility
* queues with no backlog monitoring
* external API calls with no timeout visibility
* retries with no attempt tracking
* audit logs mixed indiscriminately with debug logs
* installing excessive monitoring infrastructure without a need

---

# Minimal Implementation Standard

For a normal Express + database application, start with:

```text
1. Structured logger
2. Request ID
3. Request duration
4. Central error handler
5. Safe error responses
6. Sensitive-data redaction
7. Database connection monitoring
8. Health endpoint
9. Important security-event logging
10. Important administrative audit events
```

Then add:

```text
metrics
error tracking
queue monitoring
external-service monitoring
distributed tracing
```

only when the project benefits from them.

---

# Recommended Observability Architecture

A practical backend can follow:

```text
                    ┌─────────────────┐
                    │     Client      │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Request ID      │
                    │ Middleware       │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Authentication  │
                    │ Authorization   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │     Route       │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   Controller    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    Service      │
                    └──────┬────┬─────┘
                           │    │
                  ┌────────┘    └────────┐
                  ▼                      ▼
          ┌───────────────┐      ┌────────────────┐
          │   Database    │      │ External APIs  │
          └───────────────┘      └────────────────┘

                         │
                         ▼
                ┌─────────────────┐
                │ Observability   │
                │                 │
                │ Logs            │
                │ Metrics         │
                │ Traces          │
                │ Errors          │
                │ Audit Events    │
                └─────────────────┘
```

The exact implementation can be much simpler.

The diagram represents the relationship between the application and its observability signals rather than requiring every component.

---

# Final Principle

Observability should answer:

> **"What happened, why did it happen, and how do I prove it?"**

Do not measure everything.

Measure what helps you understand:

```text
availability
correctness
performance
security
failures
user-impacting problems
background processing
external dependencies
```

Do not turn the backend into a surveillance machine that produces mountains of useless logs.

Prefer:

**meaningful events over noisy logs**

**structured data over random strings**

**correlation over isolated messages**

**actionable metrics over vanity metrics**

**useful alerts over constant alerts**

**safe diagnostics over exposed secrets**

**appropriate complexity over unnecessary infrastructure**

A professional backend should not only work.

When something inevitably breaks, you should be able to look at the system and say:

**"I know what broke, where it broke, when it broke, what caused it, and what was affected."**

```
```
