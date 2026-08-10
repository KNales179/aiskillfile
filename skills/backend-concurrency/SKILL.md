````md
---
name: backend-concurrency
description: Design backend systems that remain correct, predictable, and responsive when multiple requests, users, processes, or workers operate on shared resources at the same time. Use this skill when implementing or reviewing concurrent database operations, race-condition prevention, transactions, atomic updates, idempotency, duplicate-request handling, resource locking, state transitions, counters, inventory-like resources, job processing, or any backend operation where simultaneous execution could produce inconsistent data.
---

# Backend Concurrency

Design backend systems with the assumption that multiple things can happen at the same time.

A backend must not assume that:

- only one user will perform an action at a time
- requests will arrive in order
- a request will only be submitted once
- a database read will still be valid when it is written back
- two workers will never process the same job
- a client will wait for the previous request before sending another
- a frontend button can only be clicked once
- an operation will always complete before another operation starts

Concurrency bugs are often difficult to reproduce because the code can appear completely correct when requests happen sequentially.

The goal of this skill is to make backend operations remain correct under concurrent execution.

Prioritize:

- data consistency
- atomicity
- correctness
- predictable state transitions
- idempotency
- safe concurrent updates
- transaction boundaries
- duplicate-request protection
- worker coordination
- resource integrity
- controlled contention
- clear failure behavior

Do not add locks, transactions, queues, or complicated synchronization mechanisms without a real concurrency problem.

Prefer the simplest mechanism that guarantees correctness.

---

# Core Principles

## Assume concurrent execution

Every backend endpoint should be designed with the assumption that it can receive multiple requests simultaneously.

For example:

```text
Request A ────────┐
                  ├──> Database
Request B ────────┘
````

Do not design the logic as if it were always:

```text
Request A
   ↓
complete
   ↓
Request B
```

The actual execution may be:

```text
Request A ── read ─────────────── write
Request B ───── read ───── write
```

This difference can produce incorrect results.

---

# Race Conditions

A race condition occurs when the correctness of an operation depends on the timing or ordering of concurrent operations.

A common example:

```text
Current balance = ₱100

Request A:
read balance → ₱100

Request B:
read balance → ₱100

Request A:
subtract ₱80

Request B:
subtract ₱80

Final balance:
₱20
```

The expected result should not allow both operations to succeed.

The problem is not necessarily that the subtraction is wrong.

The problem is that both requests operated on the same stale value.

---

## Identify race-condition opportunities

Pay particular attention to operations involving:

* balances
* counters
* inventory
* quotas
* available slots
* status changes
* ownership
* reservations
* approvals
* transactions
* payments
* driver availability
* assignment
* booking
* queue jobs
* uploads
* sequential numbers
* unique identifiers
* timestamps
* authentication state
* account status
* permission changes

Whenever multiple requests can modify the same resource, consider concurrency explicitly.

---

# Read-Modify-Write Problems

One of the most common concurrency mistakes is:

```text
read
↓
modify in application
↓
write
```

For example:

```js
const driver = await Driver.findById(id);

driver.totalTrips += 1;

await driver.save();
```

This may become unsafe if multiple requests execute the same operation simultaneously.

Two requests can both read:

```text
totalTrips = 10
```

Both calculate:

```text
10 + 1 = 11
```

Both save:

```text
11
```

The expected result was:

```text
12
```

---

# Prefer Atomic Database Operations

When the database can perform the operation atomically, prefer that approach.

For example:

```js
await Driver.updateOne(
  { _id: driverId },
  { $inc: { totalTrips: 1 } }
);
```

This allows the database to perform the increment safely without exposing the application to the same read-modify-write race.

Prefer operations such as:

* `$inc`
* `$set`
* `$unset`
* `$push`
* `$pull`
* conditional updates
* atomic find-and-update operations

when appropriate.

The exact operation depends on the database being used.

---

# Conditional Updates

When changing state, include the expected current state in the update condition whenever appropriate.

Instead of:

```js
update({
  status: "APPROVED"
});
```

consider:

```js
update(
  {
    _id: id,
    status: "PENDING"
  },
  {
    $set: {
      status: "APPROVED"
    }
  }
);
```

This prevents another request from approving a resource that has already transitioned to another state.

Conceptually:

```text
PENDING
   ↓
APPROVED
```

should not silently become:

```text
REJECTED
   ↓
APPROVED
```

because another request modified the resource between the read and the update.

---

# Check the Update Result

Do not assume that a conditional update succeeded.

Check the result.

For example:

```js
const result = await Model.updateOne(
  {
    _id: id,
    status: "PENDING"
  },
  {
    $set: {
      status: "APPROVED"
    }
  }
);

if (result.modifiedCount === 0) {
  // Resource was not updated.
  // It may no longer satisfy the expected state.
}
```

The backend should distinguish between:

```text
update succeeded
```

and:

```text
update did not happen
```

This is especially important when concurrency controls depend on the update condition.

---

# State Machines

Use explicit state transitions for resources that have meaningful lifecycle states.

For example:

```text
PENDING
   ├──> APPROVED
   └──> REJECTED
```

Instead of allowing arbitrary status changes:

```text
PENDING → APPROVED
PENDING → REJECTED
APPROVED → PENDING
APPROVED → REJECTED
REJECTED → APPROVED
```

unless those transitions are actually valid.

Define which transitions are allowed.

Example:

```js
const allowedTransitions = {
  PENDING: ["APPROVED", "REJECTED"],
  APPROVED: [],
  REJECTED: []
};
```

The exact implementation may differ, but the principle should remain.

---

# Prevent Invalid Concurrent Transitions

Consider:

```text
Request A:
approve transaction

Request B:
reject transaction
```

Both requests arrive at approximately the same time.

Without proper concurrency control:

```text
A → APPROVED
B → REJECTED
```

The final state depends on which write happens last.

Instead, make the transition conditional:

```text
Only change:

PENDING → APPROVED
```

or:

```text
Only change:

PENDING → REJECTED
```

Once one succeeds, the other should fail because the resource is no longer `PENDING`.

---

# Idempotency

Idempotency means that repeating the same operation does not unintentionally produce additional effects.

This is especially important for:

* payments
* transactions
* booking
* registration
* resource creation
* notifications
* external API calls
* webhook processing
* job processing
* file operations

A request may be repeated because of:

* network failure
* frontend retry
* user double-click
* mobile reconnection
* timeout
* reverse proxy retry
* client retry logic

Do not assume:

```text
one request = one execution
```

---

# Idempotency Keys

For operations where duplicate execution is dangerous, support idempotency keys.

Example:

```http
POST /api/transactions
Idempotency-Key: 8f3e...
```

The backend can associate the key with the result of the operation.

If the same key is received again:

```text
Request 1
   ↓
process operation
   ↓
save result

Request 2
   ↓
same idempotency key
   ↓
return existing result
```

instead of executing the operation again.

---

# Idempotency Storage

When implementing idempotency, consider storing:

* idempotency key
* user/account identifier
* operation type
* request fingerprint where appropriate
* processing status
* response/result
* creation timestamp
* expiration timestamp

Be careful about allowing the same idempotency key to be reused for a different request.

A key should generally represent one logical operation.

---

# Idempotency States

For operations that can take time, useful states may include:

```text
PROCESSING
COMPLETED
FAILED
```

For example:

```text
Request
   ↓
Idempotency key
   ↓
PROCESSING
   ↓
operation
   ↓
COMPLETED
```

A repeated request can then determine whether the original operation:

* is still running
* completed successfully
* failed
* can safely be retried

---

# Database Transactions

Use database transactions when multiple database changes must succeed or fail together.

For example:

```text
Create violation
+
Create transaction record
+
Update driver record
```

If the first two succeed but the third fails, the database may become inconsistent.

A transaction can provide:

```text
BEGIN
   ↓
Operation A
   ↓
Operation B
   ↓
Operation C
   ↓
COMMIT
```

or:

```text
BEGIN
   ↓
Operation A
   ↓
Operation B
   ↓
Operation C fails
   ↓
ROLLBACK
```

Use transactions when atomicity between multiple operations is genuinely required.

Do not place every endpoint inside a transaction automatically.

---

# Transaction Boundaries

A transaction should contain the smallest meaningful set of operations that must remain atomic.

Avoid:

```text
BEGIN TRANSACTION

database operation
↓
external API request
↓
email sending
↓
file upload
↓
long computation

COMMIT
```

Long-running external operations can hold database resources unnecessarily.

Prefer:

```text
Database transaction
   ↓
commit database state
   ↓
background job
   ↓
external operation
```

when the business logic permits it.

---

# External Services and Transactions

Do not assume that a database transaction can automatically roll back an external service.

For example:

```text
Database transaction
+
Cloudinary upload
```

If the Cloudinary upload succeeds but the database transaction rolls back, the external upload does not automatically disappear.

Similarly:

```text
Database commit
+
email sending
```

cannot guarantee that the email service will also succeed.

Treat external systems as separate failure domains.

Use appropriate patterns such as:

* background jobs
* retry mechanisms
* idempotency
* compensation
* outbox-style processing

when required.

---

# Optimistic Concurrency Control

Optimistic concurrency assumes conflicts are relatively uncommon.

The backend allows updates but verifies that the resource has not changed since it was read.

A common approach is a version field:

```text
version = 5
```

Client reads:

```text
version = 5
```

Another request updates the resource:

```text
version = 6
```

The original request attempts:

```text
update where version = 5
```

The update fails because the resource is now:

```text
version = 6
```

This prevents stale writes from silently overwriting newer data.

---

# Version Fields

A resource can maintain:

```js
version: Number
```

or use an equivalent database mechanism.

Conceptually:

```text
Read:
version 10

Update:
WHERE id = X
AND version = 10

Success:
version → 11
```

If another process already changed it:

```text
version = 11

WHERE version = 10
```

matches nothing.

The backend should then report a concurrency conflict rather than overwriting the newer state.

---

# Timestamps as Concurrency Markers

In some systems, a timestamp such as:

```text
updatedAt
```

can be used as a concurrency marker.

However, timestamps must be handled carefully.

Prefer an explicit version number when precise optimistic concurrency semantics are required.

Do not assume timestamps alone are sufficient for every concurrency problem.

---

# Pessimistic Locking

Pessimistic locking assumes conflicts are possible and prevents concurrent operations from modifying a resource simultaneously.

Conceptually:

```text
Request A
   ↓
lock resource
   ↓
modify
   ↓
commit
   ↓
unlock

Request B
   ↓
wait
   ↓
resource becomes available
   ↓
continue
```

Use this only when appropriate.

Locks can introduce:

* waiting
* contention
* deadlocks
* reduced throughput
* operational complexity

Do not use a lock when an atomic update or optimistic concurrency mechanism is sufficient.

---

# Distributed Locks

When multiple server instances or workers can modify the same resource, an in-memory lock is usually insufficient.

For example:

```text
Server A
Server B
Server C
```

A JavaScript variable such as:

```js
let locked = false;
```

only exists inside one process.

Server B does not know that Server A set:

```js
locked = true;
```

For distributed coordination, use an appropriate shared mechanism when genuinely necessary.

Possible mechanisms include:

* database-based locks
* Redis-based locks
* queue-level exclusivity
* unique database constraints
* atomic operations

Choose based on the infrastructure actually used by the project.

---

# Avoid In-Memory Concurrency Assumptions

Do not rely on:

```js
const processing = new Set();
```

as the sole protection against duplicate processing in a multi-instance deployment.

It may work for a single local server but fail when:

```text
Load Balancer
   ├── Server A
   ├── Server B
   └── Server C
```

receives requests.

The same resource may be processed simultaneously by different instances.

---

# Unique Constraints

Use database uniqueness guarantees whenever the business rule requires uniqueness.

For example:

```text
username
email
plate number
transaction reference
admin code
idempotency key
```

Do not rely solely on:

```js
const existing = await Model.findOne({ email });

if (!existing) {
  await Model.create({ email });
}
```

Two concurrent requests can both observe:

```text
no existing record
```

and both create the record.

A database-level unique constraint provides a stronger guarantee.

---

# Application Validation vs Database Constraints

Application validation improves user experience.

Database constraints protect data integrity.

Use both when appropriate.

```text
Application
   ↓
validate
   ↓
Database constraint
   ↓
persist
```

Do not assume frontend or controller validation alone guarantees uniqueness or consistency.

---

# Duplicate Requests

Expect duplicate requests.

They may originate from:

* double-clicking
* repeated form submission
* frontend retries
* mobile network instability
* browser refresh
* timeout followed by retry
* user reopening a screen
* multiple client instances

The backend should decide whether duplicate execution is:

```text
safe
```

or:

```text
dangerous
```

and handle it accordingly.

---

# Double-Submission Protection

For operations such as:

```text
Create transaction
Create violation
Submit application
Assign driver
Book ride
Process payment
```

consider:

* idempotency keys
* unique business identifiers
* state checks
* database constraints
* atomic updates

Do not rely exclusively on disabling the frontend button.

The frontend is a convenience layer.

The backend is the enforcement layer.

---

# Queue Concurrency

When background workers are introduced, concurrency becomes especially important.

For example:

```text
Queue
 ├── Job A
 ├── Job B
 ├── Job C
 └── Job D

Worker 1
Worker 2
Worker 3
```

Multiple workers may attempt to process jobs simultaneously.

The system must define:

* whether jobs may run concurrently
* maximum concurrency
* whether jobs for the same resource must be serialized
* how duplicate jobs are handled
* how failed jobs are retried
* how jobs are acknowledged
* what happens if a worker crashes

Do not assume that putting work into a queue automatically solves concurrency.

---

# Job Idempotency

Workers may execute the same logical job more than once because of:

* worker crashes
* visibility timeout expiration
* acknowledgement failure
* network failure
* retry mechanisms
* process restarts

Design jobs so repeated execution is safe whenever practical.

For example:

```text
Job:
Generate monthly report
```

should not accidentally create five copies simply because the worker retried.

Use:

* unique job identifiers
* database state checks
* idempotency keys
* atomic state transitions
* upserts
* unique constraints

where appropriate.

---

# Per-Resource Serialization

Sometimes global serialization is unnecessary.

For example:

```text
Driver A
Driver B
Driver C
```

can often be processed concurrently.

But operations for the same driver may need ordering:

```text
Driver A
   ├── status update
   ├── assignment
   └── availability update
```

A useful strategy is:

```text
same resource → serialized
different resources → concurrent
```

This can provide better throughput than processing everything sequentially.

---

# Avoid Global Locks

Do not use one global lock for unrelated resources unless absolutely necessary.

Bad:

```text
Lock entire system
   ↓
process Driver A
   ↓
unlock
```

This can unnecessarily block:

```text
Driver B
Driver C
Driver D
```

Prefer narrower concurrency boundaries.

For example:

```text
Lock Driver A
```

rather than:

```text
Lock all drivers
```

when the business rule allows it.

---

# Counters

Counters are common sources of race conditions.

Avoid:

```text
read counter
↓
counter + 1
↓
save
```

when multiple requests can update it.

Prefer atomic increments when supported.

Examples include:

* trip count
* violation count
* view count
* notification count
* available seats
* stock count
* sequence numbers

---

# Inventory and Capacity

Any resource with limited availability requires concurrency protection.

Examples:

```text
5 available slots

Request A → reserve
Request B → reserve
Request C → reserve
```

Do not rely on:

```text
if (available > 0) {
   available -= 1;
}
```

when the read and write are separate operations.

Use atomic conditional updates or appropriate transactional mechanisms.

Conceptually:

```text
UPDATE resource
SET available = available - 1
WHERE id = ?
AND available > 0
```

Then verify whether the update succeeded.

---

# Booking and Reservation

Booking systems are especially sensitive to race conditions.

A dangerous pattern is:

```text
check availability
↓
availability exists
↓
create booking
```

Two requests can both pass the availability check.

Prefer a mechanism where the reservation itself participates in the concurrency guarantee.

Possible approaches include:

* atomic state changes
* unique constraints
* database transactions
* conditional updates
* locking
* idempotency keys

Choose the simplest mechanism that satisfies the booking rules.

---

# Unique Business Identifiers

When an operation should only happen once, generate or require a unique business identifier.

Examples:

```text
transaction reference
booking reference
application number
violation reference
payment reference
job ID
```

A unique database constraint can prevent duplicate records even when requests race.

---

# Database Isolation

When using transactions, understand that transaction isolation affects what concurrent operations can observe.

Common isolation concepts include:

* Read Uncommitted
* Read Committed
* Repeatable Read
* Serializable

Do not automatically choose the strongest isolation level.

Higher isolation can increase:

* locking
* contention
* latency
* deadlock probability
* resource consumption

Choose the minimum isolation level that provides the required correctness.

---

# Deadlocks

Deadlocks can occur when concurrent operations wait for each other's resources.

Example:

```text
Transaction A
   locks Resource 1
   waits for Resource 2

Transaction B
   locks Resource 2
   waits for Resource 1
```

Neither can continue.

Avoid deadlocks by:

* keeping transactions short
* locking resources in a consistent order
* avoiding unnecessary locks
* minimizing transaction scope
* handling database deadlock errors appropriately

Do not attempt to solve deadlocks by blindly retrying everything.

---

# Consistent Lock Ordering

If multiple resources must be locked, use a consistent ordering when possible.

For example:

```text
Always lock:

Driver A
before
Driver B
```

instead of allowing:

```text
Request 1:
A → B

Request 2:
B → A
```

Consistent ordering reduces deadlock opportunities.

---

# Keep Critical Sections Small

A critical section is the part of an operation that must be protected from conflicting concurrent execution.

Keep it small.

Avoid:

```text
lock
↓
database query
↓
external API
↓
file processing
↓
email
↓
complex calculation
↓
unlock
```

Prefer:

```text
prepare data
↓
critical operation
↓
unlock
↓
external processing
```

when business rules allow.

---

# Request Cancellation

Long-running operations should not necessarily continue indefinitely after the client has disappeared.

Where supported, consider:

* request cancellation
* AbortController
* database query cancellation
* timeout propagation
* worker cancellation

However, be careful.

Cancelling a request does not necessarily mean a backend operation should be rolled back.

For example:

```text
Client disconnects
```

does not automatically mean:

```text
database transaction should disappear
```

Business logic determines whether the operation should continue.

---

# Timeouts

Every external or potentially long-running operation should have an appropriate timeout.

Avoid requests that can hang indefinitely.

Consider timeouts for:

* database operations
* HTTP calls
* third-party services
* file operations
* queue operations

A timeout should produce a controlled failure.

Do not allow an unavailable dependency to consume server resources indefinitely.

---

# Concurrency Limits

Sometimes the correct solution is to limit how much work happens simultaneously.

For example:

```text
1000 requests
        ↓
maximum 20 expensive operations
        ↓
remaining work waits
```

Useful for:

* image processing
* PDF generation
* large exports
* CPU-heavy tasks
* third-party API calls
* expensive database operations

Do not allow expensive work to consume every available server resource.

---

# Backpressure

Backpressure occurs when producers generate work faster than consumers can process it.

Example:

```text
Requests:
1000 jobs/sec

Worker capacity:
100 jobs/sec
```

Without control:

```text
queue
↓
↓
↓
↓
grows indefinitely
```

Possible strategies include:

* queue limits
* rate limiting
* concurrency limits
* rejecting excess work
* delayed processing
* batching
* load shedding

The backend should degrade predictably instead of collapsing under increasing work.

---

# Batching

When many concurrent operations target the same system, batching may reduce contention and overhead.

For example:

```text
100 individual database writes
```

may sometimes become:

```text
1 bulk operation
```

Use batching when it actually improves performance or reduces contention.

Do not batch operations when doing so makes correctness or error handling unnecessarily complicated.

---

# Atomic Upserts

When appropriate, use atomic upsert operations instead of:

```text
find
↓
if not found
↓
create
```

because two concurrent requests can both observe that the record does not exist.

An atomic upsert can combine the decision and write into a database-supported operation.

---

# Compare-and-Swap Thinking

A useful concurrency concept is:

```text
"Update this resource only if it still has the value I expect."
```

Conceptually:

```text
IF version == 5
THEN update
ELSE reject
```

This pattern is useful for:

* editing records
* status transitions
* approvals
* assignments
* resource ownership
* administrative actions

It prevents stale clients from silently overwriting newer changes.

---

# Stale Data

A client may hold stale information.

Example:

```text
Admin A opens driver record
status = REGISTERED

Admin B changes status
REGISTERED → INACTIVE

Admin A submits an old edit
```

Do not blindly allow Admin A to overwrite the newer state.

Use:

* version checking
* updated-at checks where appropriate
* conditional updates
* conflict responses

The user should receive a meaningful conflict response when necessary.

---

# HTTP Semantics and Concurrency

Use appropriate HTTP methods and status codes.

For example:

```text
GET
→ retrieval

POST
→ creation/action

PUT
→ replacement

PATCH
→ partial update

DELETE
→ deletion
```

Where concurrency conflicts occur, use an appropriate response such as:

```text
409 Conflict
```

when the request conflicts with the current resource state.

Do not return:

```text
200 OK
```

for an operation that actually failed due to a concurrency conflict.

---

# Error Handling

Concurrency failures are not necessarily server crashes.

Distinguish between:

```text
validation error
authentication error
authorization error
not found
conflict
temporary dependency failure
database failure
unexpected server failure
```

A concurrency conflict should provide a response that allows the client to understand that the resource changed.

Avoid leaking internal database details.

---

# Retry Carefully

Not every concurrency failure should be retried automatically.

Potentially retryable:

```text
temporary database deadlock
temporary network failure
transient dependency failure
```

Potentially not retryable:

```text
invalid state transition
duplicate business operation
authorization failure
validation failure
resource already processed
```

A retry that repeats a non-idempotent operation can make the problem worse.

Before retrying, determine:

1. Is the operation safe to repeat?
2. Is the failure temporary?
3. Could the original operation have actually succeeded?
4. Is an idempotency mechanism available?

---

# Retry and Idempotency

When retries are required:

```text
request
↓
operation
↓
timeout
↓
did operation succeed?
↓
retry
```

The backend may not know whether the first operation completed.

This is why idempotency is important.

Use idempotency for operations where repeating the operation could cause damage or duplication.

---

# Authentication Concurrency

Authentication systems also have concurrency concerns.

Consider:

```text
User requests password reset
User requests password reset again
```

or:

```text
Two refresh requests
   ↓
same refresh token
```

The backend should define how concurrent authentication operations behave.

Consider:

* token rotation
* refresh-token reuse detection
* session invalidation
* password changes
* account lock state
* verification state
* trusted-device state

Security-related concurrency rules belong alongside the broader authentication design.

---

# Authorization State Changes

Permissions can change while a request is being processed.

For example:

```text
Request begins
↓
User has permission
↓
Admin revokes permission
↓
Request continues
```

Do not assume authorization state is immutable during long-running operations.

For sensitive operations, enforce authorization at the appropriate point in the operation.

Do not rely exclusively on permissions checked much earlier in a long workflow.

---

# File and Upload Concurrency

Uploads can also race.

Examples:

```text
Admin A replaces receipt
Admin B replaces receipt
```

or:

```text
two workers process same uploaded document
```

Consider:

* unique file identifiers
* versioning
* ownership checks
* atomic database state changes
* cleanup of orphaned files
* idempotent processing
* background workers

Do not assume that successfully uploading a file means the database operation also succeeded.

---

# Cache Concurrency

Caches can introduce stale or inconsistent behavior.

Be careful with:

```text
database
↓
cache
```

when concurrent updates occur.

Example:

```text
Request A updates database
Request B reads old cache
Request C writes cache
```

Consider:

* cache invalidation
* cache versioning
* TTL
* write-through strategies
* stale-while-revalidate
* atomic cache operations

Do not allow cached state to override authoritative database state unintentionally.

---

# Distributed Systems

As the backend grows, concurrency may exist across:

```text
API servers
workers
databases
caches
queues
external services
```

Do not assume a single-process execution model.

A design that works on:

```text
localhost
```

may fail when deployed as:

```text
Load Balancer
   ├── API Server 1
   ├── API Server 2
   ├── API Server 3
   │
   ├── Worker 1
   └── Worker 2
```

Always identify where state lives.

---

# Source of Truth

For every important piece of state, determine the authoritative source.

Examples:

```text
Database
→ authoritative transaction state

Cache
→ temporary representation

Frontend
→ user interface state

Queue
→ pending work

Cloud storage
→ file contents
```

Do not allow two systems to independently become authoritative for the same state without a deliberate synchronization strategy.

---

# Consistency vs Responsiveness

Concurrency protection should not make every operation unnecessarily slow.

Balance:

```text
correctness
+
consistency
+
latency
+
throughput
```

Do not choose maximum locking or maximum transaction isolation simply because it feels safer.

The correct solution depends on the business requirement.

---

# Concurrency Checklist

Before implementing a potentially concurrent operation, ask:

* [ ] Can multiple requests execute this simultaneously?
* [ ] Can the same user submit it twice?
* [ ] Can different users modify the same resource?
* [ ] Can multiple backend instances execute it?
* [ ] Can a worker process it more than once?
* [ ] Is there a read-modify-write sequence?
* [ ] Can an atomic database operation solve it?
* [ ] Does the operation require a transaction?
* [ ] Does the resource need a unique constraint?
* [ ] Does it require idempotency?
* [ ] Can stale data overwrite newer data?
* [ ] Are state transitions restricted?
* [ ] Can two workers process the same resource?
* [ ] Are locks actually necessary?
* [ ] Could a lock cause contention?
* [ ] Could a deadlock occur?
* [ ] Are retries safe?
* [ ] Are timeouts defined?
* [ ] Can the operation overwhelm the server?
* [ ] What happens if the client disconnects?
* [ ] What happens if the database fails halfway through?
* [ ] What happens if an external service succeeds but the database fails?
* [ ] What happens if the backend crashes after performing the operation but before responding?

---

# Implementation Decision Guide

Use this decision process before introducing complexity.

## Simple atomic change

If the operation can be expressed as one safe database operation:

```text
Use atomic update
```

---

## Unique resource

If the requirement is:

```text
only one record may exist
```

use:

```text
database unique constraint
```

plus normal application validation.

---

## Multiple related database changes

If several changes must succeed together:

```text
use a database transaction
```

---

## Stale client updates

If an old client must not overwrite newer data:

```text
use optimistic concurrency
```

with:

```text
version
```

or an appropriate equivalent.

---

## Duplicate requests

If repeating the request must not repeat the business operation:

```text
use idempotency
```

---

## Expensive background work

If the operation does not need to complete during the HTTP request:

```text
use a queue and worker
```

---

## High contention

If many processes compete for the same resource:

```text
use appropriate atomic operations,
conditional updates,
transactions,
or locking
```

Do not immediately introduce distributed locks.

---

## External service dependency

If an operation depends on another service:

```text
use timeout
+
appropriate retry
+
idempotency
```

when appropriate.

---

# Avoid These Anti-Patterns

Do not:

* assume requests execute sequentially
* rely on frontend button disabling for correctness
* use in-memory locks as distributed locks
* perform unsafe read-modify-write operations
* rely only on application-level uniqueness checks
* retry non-idempotent operations blindly
* hold database locks while calling external services
* keep transactions open unnecessarily
* use global locks for unrelated resources
* use transactions for every single database operation
* use the strongest isolation level without justification
* assume queues automatically prevent duplicate jobs
* assume a request timeout means the operation did not execute
* assume a client disconnect means the backend should stop
* silently overwrite newer data
* ignore conditional-update failures
* treat every concurrency conflict as a server error
* introduce Redis or another distributed system when a database constraint is sufficient

---

# Practical Examples

## Unsafe counter

```js
const driver = await Driver.findById(driverId);

driver.tripCount += 1;

await driver.save();
```

Potential problem:

```text
Request A reads 10
Request B reads 10

A writes 11
B writes 11

Expected: 12
Actual:   11
```

Prefer an atomic increment where supported.

---

## Safe state transition

Instead of:

```js
const transaction = await Transaction.findById(id);

transaction.status = "APPROVED";

await transaction.save();
```

use a conditional transition:

```js
const result = await Transaction.updateOne(
  {
    _id: id,
    status: "PENDING"
  },
  {
    $set: {
      status: "APPROVED"
    }
  }
);
```

Then verify whether the transition succeeded.

---

## Unsafe uniqueness check

Avoid relying solely on:

```js
const existing = await Admin.findOne({
  username
});

if (!existing) {
  await Admin.create({
    username
  });
}
```

Two requests can pass the check simultaneously.

Use a database-level unique constraint and handle duplicate-key errors.

---

# Architecture Principle

A production backend should be designed as if this is always possible:

```text
                  ┌── Request A
                  │
Client ───────────┼── Request B
                  │
                  ├── Request C
                  │
                  └── Retry
                         │
                         ↓
                    API Server
                    ┌────┴────┐
                    ↓         ↓
                Database    Queue
                    ↓         ↓
                 Worker A   Worker B
```

Every layer may execute concurrently.

Therefore, correctness should come from explicit guarantees:

```text
Database constraints
        +
Atomic operations
        +
Transactions
        +
State validation
        +
Idempotency
        +
Controlled concurrency
        +
Appropriate locking
        +
Safe retry behavior
```

rather than assumptions about execution order.

---

# Final Principle

Concurrency is not an edge case.

It is the normal operating condition of a backend.

Design operations so that:

**two requests arriving together produce a correct result.**

When possible:

**prefer atomic database operations over application-level locking.**

When multiple changes must succeed together:

**use transactions.**

When stale clients can overwrite newer state:

**use optimistic concurrency.**

When duplicate execution is dangerous:

**use idempotency.**

When background work can run independently:

**use queues and workers.**

When contention becomes unavoidable:

**control concurrency deliberately.**

When retries are required:

**make sure the operation is safe to retry.**

When a simple database guarantee can solve the problem:

**use the database instead of adding another system.**

The objective is not to eliminate concurrency.

The objective is to make concurrency **safe, predictable, and intentional**.

```
```
