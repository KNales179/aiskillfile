````md
---
name: backend-performance
description: Build fast, efficient, scalable, and resource-conscious backend systems by optimizing request handling, database access, concurrency, asynchronous work, caching, connection management, payloads, pagination, queues, external services, CPU and memory usage, and overall system throughput. Use this skill whenever designing, implementing, reviewing, debugging, or improving backend performance. Optimize based on measured bottlenecks and actual project requirements rather than applying premature or unnecessary optimization.
---

# Backend Performance

Build backend systems that remain responsive, efficient, and reliable as request volume, data size, concurrent users, and background work increase.

Performance is not simply about making individual functions execute faster.

A performant backend should:

- respond quickly to users
- avoid unnecessary work
- use database resources efficiently
- handle concurrent requests safely
- avoid blocking the event loop
- minimize unnecessary network traffic
- process background work asynchronously when appropriate
- use caching where it provides a real benefit
- handle large datasets without excessive memory usage
- prevent slow operations from blocking unrelated requests
- degrade gracefully when dependencies become slow
- remain maintainable while being performant
- scale according to actual system requirements

The goal is:

> **Do less unnecessary work, do necessary work efficiently, and avoid making one user's work unnecessarily block another user's work.**

Do not optimize code simply because it looks inefficient.

Measure first whenever practical.

---

# Core Performance Principles

## Optimize the bottleneck, not everything

Not every part of a backend has the same performance impact.

A system may contain:

```text
Controller
Service
Database
External API
File processing
Queue
Cache
````

but only one of them may actually be responsible for the slowdown.

Before optimizing, determine:

```text
Where is the time being spent?
Why is it slow?
How frequently does it happen?
How many users are affected?
Does optimization meaningfully improve the system?
```

Avoid spending hours optimizing a function that takes `2ms` when the database query takes `900ms`.

---

# Measure Before Optimizing

When possible, measure:

* request latency
* database query duration
* external API duration
* queue wait time
* job processing time
* CPU usage
* memory usage
* event-loop delay
* request volume
* error rate
* cache hit rate

Use measurements to identify actual bottlenecks.

Do not rely entirely on intuition.

---

# User-Perceived Performance

Backend performance ultimately affects the user experience.

Prioritize operations that users directly experience:

```text
login
loading dashboard
search
fetching records
creating records
updating records
uploading files
generating reports
```

A technically optimized background task is less important than a frequently used API endpoint that takes several seconds.

---

# Performance Budgets

For important endpoints, establish reasonable expectations.

For example:

```text
simple read endpoint
→ usually fast

authentication
→ should not unnecessarily perform expensive work

search
→ should remain responsive

large report
→ may require asynchronous processing
```

Do not blindly impose the same latency target on every endpoint.

A simple:

```text
GET /drivers/:id
```

and a:

```text
POST /reports/generate
```

have fundamentally different performance characteristics.

---

# Request Lifecycle

Understand where time is spent:

```text
Request
   ↓
Middleware
   ↓
Authentication
   ↓
Authorization
   ↓
Validation
   ↓
Controller
   ↓
Service
   ↓
Database / External API
   ↓
Response
```

A request that takes `1000ms` does not necessarily have a slow controller.

It could be:

```text
authentication = 20ms
validation = 2ms
controller = 5ms
database = 850ms
external API = 120ms
response = 3ms
```

Optimize the actual expensive stage.

---

# Avoid Blocking the Node.js Event Loop

Node.js is highly dependent on the event loop.

Avoid expensive synchronous operations in request handlers.

Be careful with:

```js
fs.readFileSync()
fs.writeFileSync()
```

and CPU-heavy synchronous operations.

Large synchronous workloads can prevent other requests from being processed.

---

# Event Loop Blocking

A backend can appear to have:

```text
CPU
Database
Network
```

working normally while still becoming unresponsive because JavaScript execution is blocking the event loop.

Potential causes include:

* large loops
* expensive JSON processing
* image processing
* cryptographic computation
* large data transformations
* synchronous filesystem operations
* inefficient algorithms

Move expensive work away from the request path when appropriate.

---

# CPU-Heavy Work

If an operation is computationally expensive, consider:

* worker threads
* background jobs
* dedicated workers
* external processing services
* precomputation
* caching

Do not automatically move everything into workers.

Use them when CPU work would otherwise block important request processing.

---

# Background Jobs

Long-running operations should not necessarily remain inside an HTTP request.

Examples:

```text
report generation
large exports
image processing
document processing
email batches
data synchronization
large file conversion
```

Instead of:

```text
POST /report
     ↓
generate report for 30 seconds
     ↓
response
```

consider:

```text
POST /report
     ↓
create job
     ↓
return job ID
     ↓
worker processes job
     ↓
client checks or receives status
```

---

# Queue-Based Processing

Queues are useful when work can be processed asynchronously.

Typical architecture:

```text
API
 ↓
Queue
 ↓
Worker
 ↓
Database / External Service
```

This allows request handling and background processing to be separated.

Use queues when they solve a real workload problem.

Do not introduce a queue simply because "scalable systems use queues."

---

# Queue Backpressure

When incoming work is faster than workers can process it:

```text
Incoming:
████████████████████

Processing:
████
```

the queue grows.

Observe:

* queue depth
* oldest job age
* processing rate
* failure rate
* retry rate

If backlog continually increases, the system needs attention.

Potential solutions include:

* more workers
* faster processing
* batching
* rate limiting
* prioritization
* reducing unnecessary work

---

# Concurrency

Concurrency allows multiple operations to progress without unnecessarily waiting for one another.

Avoid accidental serialization.

Bad pattern:

```js
await operationA();
await operationB();
await operationC();
```

when the operations are completely independent.

Potentially better:

```js
await Promise.all([
  operationA(),
  operationB(),
  operationC()
]);
```

Only do this when:

* operations are independent
* concurrent execution is safe
* downstream services can handle the load

Do not blindly convert every sequential operation to `Promise.all()`.

---

# Concurrency Limits

Unlimited concurrency can be just as bad as insufficient concurrency.

Avoid:

```js
await Promise.all(hugeArray.map(processItem));
```

for thousands of expensive operations.

This can overwhelm:

* database connections
* APIs
* memory
* CPU
* file descriptors
* network resources

Use controlled concurrency when processing large collections.

---

# Controlled Concurrency

Instead of:

```text
10000 tasks
↓
10000 simultaneous operations
```

use:

```text
10000 tasks
↓
10 or 20 at a time
↓
next batch
```

The correct concurrency limit depends on:

* workload
* database capacity
* external service limits
* server resources
* expected traffic

---

# Database Performance

The database is frequently one of the most important backend performance bottlenecks.

Optimize:

* queries
* indexes
* projections
* pagination
* aggregation
* connection usage
* document size
* data modeling

Do not optimize database code blindly.

Measure query performance where possible.

---

# Avoid N+1 Queries

A common performance problem:

```text
Get 100 drivers
    ↓
Query vehicle for driver 1
Query vehicle for driver 2
Query vehicle for driver 3
...
Query vehicle for driver 100
```

This can produce:

```text
1 + 100 queries
```

instead of a more efficient approach.

Consider:

* joins where supported
* aggregation
* batching
* preloading
* appropriate denormalization

The exact solution depends on the database.

---

# MongoDB Considerations

For MongoDB applications, pay attention to:

* indexes
* query filters
* projections
* aggregation pipelines
* document size
* pagination
* collection growth
* query execution plans

Do not create indexes for every field.

Indexes improve reads but can increase:

* storage
* write cost
* memory usage
* index maintenance

Create indexes based on actual query patterns.

---

# Database Indexes

Index fields that are frequently used for:

* filtering
* sorting
* lookups
* uniqueness constraints

For example, if the application frequently searches:

```text
driverCode
plateNumber
username
email
```

appropriate indexes may be valuable.

However, determine the actual query patterns before adding indexes.

---

# Compound Indexes

When queries frequently filter by multiple fields, consider compound indexes.

For example:

```text
status + createdAt
```

may be more useful than two unrelated indexes depending on the query pattern.

Index order matters.

Design indexes around actual queries.

---

# Index Overuse

Do not create:

```text
index on everything
```

Excessive indexes can make writes slower and consume additional resources.

Each index should have a reason.

---

# Query Projection

Do not retrieve large documents when only a few fields are required.

Instead of fetching:

```text
entire user document
```

when you only need:

```text
name
role
status
```

select only the necessary fields.

This reduces:

* database work
* network transfer
* serialization
* memory usage

---

# Large Documents

Avoid unnecessarily large documents.

Large records can increase:

* read latency
* network transfer
* memory usage
* serialization cost

Do not embed huge files directly into normal database documents when an object-storage solution is more appropriate.

For example:

```text
MongoDB
→ metadata

Cloudinary / object storage
→ actual image/document
```

when appropriate.

---

# Database Pagination

Never assume a dataset will remain small forever.

Avoid endpoints that always return:

```text
all drivers
all violations
all transactions
all users
```

Use pagination for large collections.

Example:

```text
GET /drivers?page=1&limit=20
```

or cursor-based pagination where appropriate.

---

# Offset Pagination

Traditional pagination:

```text
page=1
limit=20
```

can be simple and appropriate for many administrative systems.

It is easy to understand and implement.

However, large offsets can become less efficient depending on the database and query.

Use it when the dataset and access pattern make it appropriate.

---

# Cursor Pagination

Cursor-based pagination can be better for:

* large datasets
* continuously changing data
* infinite scrolling
* high-volume feeds

Example:

```text
GET /transactions?cursor=abc123&limit=20
```

Do not introduce cursor pagination when normal page-based pagination is already sufficient.

---

# Pagination Limits

Never allow unlimited client-controlled limits.

Avoid:

```text
?limit=999999999
```

Set a maximum.

Example:

```text
default = 20
maximum = 100
```

The exact values depend on the application.

---

# Sorting

Allow only supported sorting fields.

Do not blindly pass arbitrary client-provided field names into database queries.

Use a whitelist:

```text
createdAt
name
status
```

This provides both security and predictable performance.

---

# Search Performance

Search endpoints can become expensive quickly.

Consider:

* indexes
* appropriate search strategies
* debouncing on the frontend
* minimum search length
* pagination
* result limits
* caching when appropriate

Avoid executing expensive wildcard searches against huge collections for every keystroke.

---

# Database Connection Pooling

Use connection pooling appropriately.

Creating a new database connection for every request can be inefficient.

Prefer:

```text
Application
    ↓
Connection Pool
    ↓
Database
```

rather than:

```text
Request
    ↓
new database connection
    ↓
query
    ↓
close connection
```

Follow the database driver's recommended connection management.

---

# Connection Pool Size

Do not assume:

```text
more connections = more performance
```

Too many connections can overwhelm the database.

Too few can cause unnecessary waiting.

Tune the pool based on:

* expected concurrency
* database capacity
* workload
* deployment environment

---

# Database Transactions

Transactions provide correctness but can introduce overhead.

Use transactions when atomicity is actually required.

Do not wrap unrelated operations in unnecessarily large transactions.

Long-running transactions can:

* hold resources
* increase contention
* delay other operations

Keep transactions focused.

---

# Avoid Long Transactions

Prefer:

```text
begin
 ↓
required operations
 ↓
commit
```

over:

```text
begin
 ↓
API request
 ↓
external service
 ↓
file upload
 ↓
complex computation
 ↓
database operations
 ↓
commit
```

Do not hold database transactions open while waiting on external services unless the architecture explicitly requires it.

---

# External API Performance

External services can dominate request latency.

For example:

```text
API = 30ms
Database = 50ms
External service = 1500ms
```

The external service is the bottleneck.

Consider:

* timeouts
* caching
* asynchronous processing
* batching
* parallel requests
* retry policies
* circuit breakers
* fallback behavior

---

# External API Timeouts

Never allow external calls to wait indefinitely.

Set reasonable timeouts.

A request should not remain open forever because a dependency stopped responding.

---

# Parallel External Requests

If independent external operations can safely execute concurrently:

```js
await Promise.all([
  serviceA(),
  serviceB()
]);
```

may reduce total latency.

Sequential:

```text
A = 500ms
B = 500ms

total ≈ 1000ms
```

Parallel:

```text
A = 500ms
B = 500ms

total ≈ 500ms
```

Only use this when the operations are genuinely independent and the downstream services can handle concurrent traffic.

---

# External Service Batching

If an external API supports batch requests:

```text
100 individual requests
```

may sometimes be replaced with:

```text
10 batch requests
```

or:

```text
1 batch request
```

This can reduce:

* network overhead
* connection overhead
* request latency
* rate-limit pressure

Use batching when supported and appropriate.

---

# Caching

Caching can significantly improve performance when data:

* is expensive to calculate
* is frequently requested
* does not change constantly

Potential cache targets:

```text
configuration
reference data
public metadata
frequently accessed records
expensive calculations
external API responses
```

Do not cache everything.

---

# Cache Correctness

Caching introduces a new problem:

> How do we know the cached value is still correct?

Before adding a cache, define:

```text
What is cached?
How long is it valid?
When is it invalidated?
What happens if it is missing?
What happens if stale data is returned?
```

A fast incorrect system is still incorrect.

---

# Cache-Aside

A common strategy:

```text
Request
 ↓
Check cache
 ↓
Cache hit → return
 ↓
Cache miss
 ↓
Fetch database
 ↓
Store cache
 ↓
Return
```

This can be appropriate for frequently read data.

---

# Cache TTL

Use expiration when appropriate.

For example:

```text
configuration → longer TTL
frequently changing data → shorter TTL
real-time data → little or no caching
```

The TTL should be based on how stale the data is allowed to become.

---

# Cache Invalidation

Cache invalidation should be explicit.

When important data changes:

```text
Update database
    ↓
Invalidate/update cache
```

Do not allow stale data to remain indefinitely.

---

# Cache Stampede

If a cached value expires and many requests arrive simultaneously:

```text
Request 1 → cache miss
Request 2 → cache miss
Request 3 → cache miss
...
Request 100 → cache miss
```

all requests may hit the database simultaneously.

This can create a cache stampede.

Possible strategies include:

* request coalescing
* locking
* staggered expiration
* stale-while-revalidate
* prewarming

Use these only when the workload justifies them.

---

# Request Deduplication

If multiple identical operations are happening at the same time, consider whether they can share work.

For example:

```text
10 requests
    ↓
same expensive operation
```

could potentially become:

```text
10 requests
    ↓
1 shared operation
    ↓
10 responses
```

This is especially useful for expensive reads or external requests.

---

# Cache vs Database

Do not use caching to hide a fundamentally inefficient database query.

First ask:

```text
Can the query be improved?
Is the correct index present?
Is too much data being returned?
Is pagination missing?
```

Caching should complement a good data-access strategy.

---

# Response Payload Size

Large responses increase:

* serialization time
* network transfer
* memory usage
* client processing

Return only what the client needs.

Avoid:

```text
GET /drivers
```

returning massive nested structures when the UI only needs:

```text
id
name
plateNumber
status
```

---

# API Response Design

Performance should not come at the expense of predictable API contracts.

Avoid returning inconsistent response structures simply to save a few bytes.

Optimize meaningful payload size, not readability to the point of breaking maintainability.

---

# Compression

For larger textual responses, HTTP compression may reduce network transfer.

Potentially compress:

```text
JSON
HTML
CSS
JavaScript
text
```

depending on the architecture.

Do not compress already compressed content unnecessarily.

Examples:

```text
JPEG
PNG
WebP
ZIP
```

---

# Compression Tradeoffs

Compression uses CPU.

A system should balance:

```text
CPU cost
vs
network savings
```

For large JSON responses, compression can often be valuable.

For tiny responses, compression may provide little benefit.

---

# File Uploads

Large file uploads should not necessarily pass through unnecessary processing layers.

Avoid:

```text
Client
 ↓
API
 ↓
load entire file into memory
 ↓
process
 ↓
upload elsewhere
```

when a streaming or direct-upload architecture is appropriate.

---

# Streaming

For large data, streaming can prevent loading everything into memory at once.

Instead of:

```text
Read entire 2GB file
 ↓
Store in memory
 ↓
Process
```

prefer:

```text
Read chunk
 ↓
Process chunk
 ↓
Read next chunk
```

when supported.

---

# Streaming Responses

Streaming can be useful for:

* large exports
* generated files
* long-running processes
* server-sent events
* AI-generated output
* progress updates

Do not use streaming simply because it sounds modern.

Use it when incremental delivery improves the experience or resource usage.

---

# Memory Efficiency

Avoid unnecessarily loading huge datasets into memory.

Bad:

```js
const allDrivers = await Driver.find({});
```

when there are potentially hundreds of thousands of records.

Better approaches may include:

* pagination
* cursor iteration
* streaming
* aggregation
* batch processing

---

# Large Data Processing

When processing large collections:

```text
Do not:
load everything
↓
process everything
↓
save everything
```

Consider:

```text
batch
↓
process
↓
release memory
↓
next batch
```

This keeps memory usage more predictable.

---

# Batching

Batch operations when appropriate.

For example:

```text
1000 individual writes
```

may be more efficiently performed as:

```text
10 batches of 100
```

depending on the database and operation.

Do not make batches so large that they create:

* memory spikes
* long transactions
* oversized requests
* difficult failure recovery

---

# Bulk Operations

Use database bulk operations when they meaningfully reduce overhead.

Examples:

```text
bulk insert
bulk update
bulk write
```

Consider partial failure handling carefully.

Bulk operations can make retries more complicated.

---

# Avoid Excessive Serialization

Large objects can be expensive to:

```text
serialize
deserialize
clone
log
transform
```

Avoid repeatedly converting the same large object between formats.

---

# JSON Performance

Be cautious when handling huge JSON payloads.

Large JSON documents can increase:

* parsing cost
* memory usage
* response time

Prefer pagination or streaming when the dataset is large.

---

# Middleware Performance

Middleware runs frequently.

Avoid putting expensive work into global middleware unless every request needs it.

For example:

```text
app.use(expensiveOperation)
```

means every applicable request pays the cost.

Prefer route-specific middleware when appropriate.

---

# Authentication Performance

Security should not be weakened for performance.

Authentication operations can be computationally expensive by design.

For example:

```text
password hashing
```

should remain appropriately expensive.

Do not reduce password hashing cost simply to improve benchmark results.

Instead optimize surrounding operations.

---

# Authorization Performance

Authorization checks should be efficient but must remain correct.

Avoid repeatedly querying the database for the same permission information within one request if it can safely be reused.

However, do not cache authorization state so aggressively that permission changes become unsafe.

---

# Token Verification

Token verification should remain efficient and secure.

Avoid unnecessary repeated verification within the same request lifecycle.

If the authenticated identity is already securely established by middleware, downstream controllers should reuse that validated context rather than repeatedly performing the same work.

---

# Rate Limiting

Rate limiting protects performance as well as security.

Without rate limiting:

```text
legitimate traffic
+
abusive traffic
+
bots
+
accidental request storms
```

can consume backend resources.

Apply rate limits where appropriate, especially to expensive endpoints.

---

# Expensive Endpoint Protection

Endpoints that trigger expensive operations may need stricter controls.

Examples:

```text
report generation
file conversion
search
bulk operations
password reset
email sending
```

Consider:

* rate limiting
* authentication
* authorization
* queues
* concurrency limits
* caching

---

# Request Cancellation

Do not continue expensive work when the client has already abandoned the request if the operation is safe to cancel.

Potential examples:

```text
large external API call
database operation
streaming response
long computation
```

Cancellation support depends on the libraries involved.

Do not introduce cancellation complexity where the operation is already extremely short.

---

# Timeouts

Every expensive external dependency should have an intentional timeout.

Potential layers include:

```text
HTTP timeout
database timeout
queue timeout
job timeout
external API timeout
```

Avoid requests that can hang indefinitely.

---

# Retry and Performance

Retries can improve reliability but can also increase load.

For example:

```text
100 requests
 ↓
each retries 3 times
 ↓
up to 400 requests
```

This can make an outage worse.

Use:

* exponential backoff
* jitter
* maximum attempts
* retryable-error classification

only where appropriate.

---

# Retry Only Retryable Failures

Do not retry errors such as:

```text
invalid input
authentication failure
authorization failure
resource does not exist
```

Retry transient failures such as:

```text
temporary timeout
temporary network failure
temporary service unavailability
```

depending on the operation.

---

# Exponential Backoff

Avoid:

```text
retry immediately
retry immediately
retry immediately
```

Prefer increasing delays:

```text
100ms
200ms
400ms
800ms
```

with jitter where appropriate.

The exact values depend on the workload.

---

# Avoid Retry Storms

When many workers retry at exactly the same time:

```text
service fails
 ↓
all workers retry
 ↓
service overloaded
 ↓
all workers retry again
```

Use jitter and sensible retry limits.

---

# Database Performance vs Correctness

Never sacrifice:

* data integrity
* authorization
* transactional correctness
* validation
* security

merely to gain small performance improvements.

A fast backend that corrupts data is not performant in any meaningful engineering sense.

---

# API Architecture

Keep responsibilities separated when useful:

```text
Route
 ↓
Controller
 ↓
Service
 ↓
Repository / Data Access
 ↓
Database
```

This separation is not automatically a performance optimization.

Its primary benefit is maintainability.

Optimize the actual expensive layer rather than collapsing architecture merely to reduce file count.

---

# Avoid Premature Microservices

Do not split a backend into multiple services purely for performance.

A modular monolith can often handle substantial workloads.

Start with:

```text
one backend
```

and separate services only when there is a real architectural reason such as:

* independent scaling
* isolation
* team boundaries
* deployment requirements
* workload specialization

---

# Horizontal Scaling

When traffic increases, consider running multiple backend instances:

```text
             Load Balancer
              /    |    \
             /     |     \
          API 1   API 2   API 3
```

This can improve:

* throughput
* availability
* fault tolerance

but requires appropriate handling of:

* shared state
* sessions
* queues
* caches
* database connections

---

# Stateless APIs

For horizontally scaled APIs, prefer stateless request handling where practical.

Avoid storing important session state only in one server's memory.

Instead use shared infrastructure when needed:

```text
database
distributed cache
token-based authentication
shared session store
```

---

# In-Memory Cache Considerations

An in-memory cache is fast but local.

If:

```text
API 1
cache = value A

API 2
cache = missing
```

different instances can have different state.

This is important when horizontally scaling.

Use distributed caching when shared cache state is actually required.

---

# Database as a Bottleneck

Scaling API instances does not automatically solve database limitations.

For example:

```text
1 API instance
→ 100 DB operations

5 API instances
→ potentially 500 DB operations
```

More application servers can increase database pressure.

Always consider the entire system.

---

# Connection Multiplication

If each backend instance creates a database pool:

```text
10 instances
×
50 connections
=
500 database connections
```

This can overwhelm the database.

Connection pool sizing must consider total deployment capacity, not only one server.

---

# Load Testing

When performance matters, test realistic workloads.

Simulate:

```text
concurrent users
request rates
large datasets
slow dependencies
database load
background jobs
```

Do not rely only on:

```text
"I tested it once and it felt fast."
```

---

# Load Testing Scenarios

Useful scenarios include:

```text
normal load
peak load
sustained load
sudden traffic spike
large dataset
slow database
slow external API
queue backlog
```

This reveals different bottlenecks.

---

# Stress Testing

Stress testing determines how the system behaves beyond normal capacity.

Look for:

```text
latency degradation
error rates
memory growth
CPU saturation
database exhaustion
queue growth
crashes
```

The goal is not necessarily to make the system survive unlimited traffic.

The goal is to understand its limits.

---

# Performance Under Failure

A performant backend should not collapse immediately when dependencies slow down.

For example:

```text
Email service becomes slow
```

should not necessarily make:

```text
GET /drivers
```

slow if the driver endpoint does not depend on email.

Isolate unrelated work.

---

# Bulkhead Principle

Separate resource pools where appropriate.

For example:

```text
Normal API requests
        ↓
normal resources

Report generation
        ↓
background resources
```

This prevents expensive workloads from consuming all available resources.

---

# Request Prioritization

Not all work has equal urgency.

Potentially prioritize:

```text
user-facing request
```

over:

```text
large background export
```

when resource contention occurs.

Queue systems may support priorities when needed.

Do not introduce priority systems unless the workload benefits from them.

---

# Resource Limits

Set reasonable limits for:

* request body size
* file size
* pagination size
* concurrent jobs
* queue depth
* processing duration
* external request timeouts

Limits protect both performance and reliability.

---

# Memory Leaks

Watch for memory that continually grows over time.

Potential causes include:

* unbounded caches
* retained event listeners
* global arrays
* unresolved promises
* long-lived references
* accumulating job data

A backend that starts fast and becomes slower after several hours may have a memory-related problem.

---

# Unbounded Collections

Avoid structures that grow forever:

```js
const cache = {};
const history = [];
const activeJobs = [];
```

without a cleanup strategy.

Every long-lived collection needs an intentional lifecycle.

---

# Caching Large Objects

Caching a huge object may make requests faster while creating memory pressure.

Consider:

```text
cache size
TTL
frequency of use
object size
eviction policy
```

A cache is not free memory.

---

# Garbage Collection

In Node.js, excessive allocation can increase garbage collection pressure.

Avoid unnecessary creation of huge temporary objects.

For example, repeatedly creating massive transformed datasets can cause:

```text
high memory
GC pressure
latency spikes
```

Prefer efficient data processing when large workloads are involved.

---

# Algorithmic Efficiency

Use appropriate algorithms and data structures.

Be aware of complexity:

```text
O(1)
O(log n)
O(n)
O(n log n)
O(n²)
```

Avoid unnecessary `O(n²)` operations on large datasets.

Example:

```js
for (const user of users) {
  for (const role of roles) {
    ...
  }
}
```

may become expensive as both collections grow.

Consider using maps or sets when appropriate.

---

# Do Not Micro-Optimize

Do not spend significant effort changing:

```text
tiny string operation
minor variable allocation
small syntax differences
```

while the actual bottleneck is:

```text
database query
external API
network transfer
algorithmic complexity
```

Focus on high-impact changes.

---

# Performance and Maintainability

Do not turn readable code into unreadable code for a tiny performance gain.

Prefer:

```text
clear
measurable
maintainable
```

optimization.

If an optimization significantly increases complexity, document why it exists.

---

# Performance Comments

When an unusual optimization is intentional, explain:

```text
what problem it solves
why the simpler implementation was insufficient
what tradeoff it introduces
```

For example:

```js
// Batch writes to avoid creating thousands of individual
// database operations during bulk imports.
```

Avoid comments that merely restate the code.

---

# Performance Regression Prevention

After optimizing something important, preserve the improvement through:

* benchmarks
* load tests
* monitoring
* metrics
* performance budgets

Otherwise future changes may accidentally reintroduce the bottleneck.

---

# Endpoint Performance Checklist

For a slow endpoint, investigate in this order:

```text
1. Request volume
2. Middleware
3. Authentication
4. Authorization
5. Controller
6. Service logic
7. Database queries
8. External services
9. Serialization
10. Response size
```

Measure each stage when practical.

---

# Database Performance Checklist

Check:

* [ ] Appropriate indexes exist.
* [ ] Queries return only required data.
* [ ] N+1 queries are avoided.
* [ ] Large datasets are paginated.
* [ ] Query limits exist.
* [ ] Sorting is controlled.
* [ ] Connection pooling is configured appropriately.
* [ ] Connection count matches deployment scale.
* [ ] Transactions are appropriately scoped.
* [ ] Large documents are avoided.
* [ ] Slow queries can be identified.
* [ ] Bulk operations are used where appropriate.
* [ ] Database work is not unnecessarily repeated.

---

# API Performance Checklist

Check:

* [ ] Endpoints do not perform unnecessary work.
* [ ] Large responses are paginated.
* [ ] Response payloads are appropriate.
* [ ] Expensive operations can be asynchronous.
* [ ] External calls have timeouts.
* [ ] Independent calls can run concurrently when safe.
* [ ] Retry behavior is controlled.
* [ ] Expensive endpoints have appropriate limits.
* [ ] Request bodies have size limits.
* [ ] Compression is considered where beneficial.
* [ ] Request cancellation is considered for long operations.
* [ ] Performance can be measured.

---

# Background Processing Checklist

If queues or workers exist:

* [ ] Long-running tasks are not unnecessarily tied to HTTP requests.
* [ ] Queue depth is observable.
* [ ] Worker concurrency is controlled.
* [ ] Jobs have timeouts where appropriate.
* [ ] Failed jobs are visible.
* [ ] Retries are limited.
* [ ] Backoff is used for transient failures.
* [ ] Retry storms are prevented.
* [ ] Large workloads are processed in batches.
* [ ] Job payloads do not unnecessarily consume memory.
* [ ] Workers do not starve user-facing API resources.

---

# Performance Anti-Patterns

Avoid:

* optimizing without measuring
* `Promise.all()` over thousands of expensive operations
* loading entire collections into memory
* returning unlimited datasets
* creating a database connection per request
* creating excessive database connections
* querying inside loops unnecessarily
* N+1 database queries
* calling slow external services synchronously when asynchronous processing is appropriate
* retrying every failure
* retrying without backoff
* retry storms
* caching everything
* caching without invalidation rules
* unbounded in-memory caches
* unlimited request sizes
* unlimited pagination limits
* CPU-heavy synchronous work inside request handlers
* large synchronous file operations
* unnecessary transactions
* long-running transactions
* excessive middleware
* unnecessary microservices
* optimizing code that is not a bottleneck
* sacrificing correctness for speed
* sacrificing security for performance
* sacrificing maintainability for insignificant gains

---

# Practical Performance Architecture

A scalable backend may eventually resemble:

```text
                         ┌──────────────┐
                         │    Client    │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ Load Balancer│
                         └──────┬───────┘
                                │
                  ┌─────────────┼─────────────┐
                  ▼             ▼             ▼
             ┌────────┐    ┌────────┐    ┌────────┐
             │ API 1  │    │ API 2  │    │ API 3  │
             └───┬────┘    └───┬────┘    └───┬────┘
                 │             │             │
                 └─────────────┼─────────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
                    ▼                     ▼
              ┌───────────┐         ┌───────────┐
              │   Cache   │         │  Database │
              └───────────┘         └───────────┘
                    │
                    │
                    ▼
              ┌───────────┐
              │   Queue   │
              └─────┬─────┘
                    │
             ┌──────┼──────┐
             ▼      ▼      ▼
          Worker  Worker  Worker
             │      │      │
             └──────┼──────┘
                    │
                    ▼
             External Services
```

This is an example architecture, not a requirement.

A small backend may simply be:

```text
React Native
    ↓
Express
    ↓
MongoDB
```

and that can be perfectly appropriate.

---

# Scaling Strategy

When the backend becomes slow, investigate in this order:

```text
1. Remove unnecessary work
2. Optimize inefficient algorithms
3. Optimize database queries
4. Add appropriate indexes
5. Reduce payload sizes
6. Add caching where useful
7. Parallelize independent operations
8. Add concurrency limits
9. Move expensive work to queues
10. Scale workers
11. Scale API instances
12. Introduce more complex architecture only when necessary
```

Do not jump immediately to:

```text
microservices
Kubernetes
distributed cache
multiple databases
```

because an endpoint takes 800ms.

---

# Performance Decision Framework

Before adding a performance optimization, ask:

### 1. What problem does this solve?

If the answer is unclear, do not add it.

### 2. What is the measured bottleneck?

If there is no evidence, investigate first.

### 3. How often does the bottleneck occur?

A rare slow operation may not justify significant complexity.

### 4. How many users are affected?

Prioritize high-impact problems.

### 5. What resources are being consumed?

Consider:

```text
CPU
memory
database
network
external APIs
worker capacity
```

### 6. What tradeoffs does the optimization introduce?

For example:

```text
cache
→ faster reads
→ stale-data risk

parallel requests
→ lower latency
→ higher concurrency

queue
→ responsive API
→ more infrastructure complexity

large batch
→ fewer requests
→ higher memory usage
```

### 7. Can the improvement be measured?

If possible, benchmark before and after.

---

# Final Performance Principle

A professional backend is not the backend with the most optimization techniques.

It is the backend that uses its resources intelligently.

Prefer:

**measured performance over assumptions**

**efficient queries over unnecessary caching**

**controlled concurrency over unlimited parallelism**

**background processing over blocking requests**

**small payloads over unnecessary data**

**appropriate indexes over indexing everything**

**timeouts over hanging requests**

**backoff over retry storms**

**bounded resources over unlimited workloads**

**simple architecture over premature complexity**

**correctness and security over benchmark numbers**

The objective is not:

> "Make everything as fast as possible."

The objective is:

> **Make the system as fast, responsive, reliable, and scalable as the project actually requires — without turning the backend into an unnecessarily complicated machine.**

```
```
