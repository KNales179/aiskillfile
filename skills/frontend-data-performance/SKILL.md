````md
---
name: frontend-data-performance
description: Build responsive, efficient, resilient frontend data flows using request deduplication, optimistic updates, stale-while-revalidate, smart polling, retry with backoff, streaming UI, caching, request cancellation, pagination, prefetching, debouncing, throttling, and related data-fetching patterns. Use these techniques only when they provide a real benefit to the project's requirements, UX, network behavior, data freshness, or performance.
---

# Frontend Data Performance

Build frontend data flows that feel fast, responsive, reliable, and intentional.

Use advanced data-fetching and synchronization patterns when they solve an actual problem.

Do not implement every pattern by default.

A simple screen that fetches one resource once does not need:

- polling
- optimistic state management
- streaming
- complicated cache invalidation
- retry systems
- multiple layers of caching

Prefer the simplest architecture that provides the required behavior.

The objective is:

- responsive interactions
- efficient network usage
- correct data
- predictable synchronization
- graceful failure
- minimal unnecessary requests
- minimal unnecessary state complexity
- good behavior on slow or unreliable networks

---

# Core Principles

## 1. Prefer user-perceived performance

Optimize what the user experiences.

Prioritize:

- immediate visual feedback
- fast first meaningful content
- stable layouts
- predictable loading states
- minimal unnecessary blocking
- graceful recovery
- useful progress indication

Do not optimize network requests at the expense of a confusing interface.

A technically efficient application can still feel slow if:

- the screen stays blank while data loads
- every small update triggers a full-page spinner
- buttons provide no feedback
- existing data disappears during refresh
- users cannot tell whether an action succeeded
- animations unnecessarily delay interaction

Performance is not only about milliseconds.

It is also about **how quickly the interface communicates that something is happening**.

---

## 2. Avoid unnecessary requests

Before adding a request, determine:

- Is the data already available?
- Is the same request already running?
- Is cached data still usable?
- Does the data actually need to be fresh?
- Can multiple requests be combined?
- Can the request be delayed until needed?
- Can it be prefetched?
- Can it be cancelled?
- Can the UI use existing data temporarily?

Do not fetch simply because a component rendered.

---

## 3. Do not over-engineer

Use the smallest appropriate pattern.

Examples:

```text
Static content
→ no data-fetching optimization required

Simple one-time API request
→ basic loading/error handling

Normal CRUD
→ caching + deduplication where useful

Search
→ debounce + cancellation

Frequently changing dashboard
→ polling or streaming when justified

Slow generation/process
→ streaming or progress updates

Unreliable network
→ retry with backoff

Large dataset
→ pagination or infinite loading
````

Do not introduce a complex architecture merely because a technique is considered modern.

---

## 4. Correctness comes before performance

Never optimize the frontend in a way that causes incorrect data.

Prefer:

```text
correct data
+
reasonable performance
```

over:

```text
fast but incorrect data
```

Examples of unacceptable optimization:

* showing stale permissions as if they were current
* showing cached data belonging to another user
* assuming an optimistic update succeeded permanently
* hiding a failed mutation
* displaying outdated financial information without appropriate indication
* skipping a required authorization check because data was previously loaded

The backend remains the authority for protected operations.

---

## 5. Server state and UI state are different

Distinguish between local interface state and remote server state.

### Local UI state

Examples:

* modal visibility
* selected tab
* dropdown state
* input value
* temporary form values
* animation state
* sidebar visibility
* current step in a form

### Server state

Examples:

* users
* drivers
* vehicles
* transactions
* violations
* dashboard statistics
* notifications
* remote configuration
* API results

Server state may require:

* caching
* synchronization
* invalidation
* deduplication
* background refresh
* retry behavior

Do not create complicated server-state management for simple local state.

---

# Data Fetching Strategy

## 1. Start with the simplest request

A basic request should generally follow:

```text
request
↓
loading
↓
success / empty / error
```

Do not automatically add:

```text
cache
+
retry
+
polling
+
optimistic state
+
streaming
+
prefetching
```

unless the requirements justify them.

---

## 2. Determine the data characteristics

Before selecting a strategy, determine:

* How frequently does the data change?
* How important is freshness?
* How expensive is the request?
* How large is the response?
* Is the data public or private?
* Is the data user-specific?
* Can stale data be displayed?
* Can the request fail temporarily?
* Is the operation reversible?
* Is the operation idempotent?
* Does the user expect immediate feedback?
* Does the user need live updates?
* Is the user likely to request the same data again?

These characteristics should determine the implementation.

---

# Request Deduplication

## Purpose

Request deduplication prevents identical requests from being made unnecessarily at the same time.

Instead of:

```text
Component A → GET /api/drivers
Component B → GET /api/drivers
Component C → GET /api/drivers
```

prefer sharing the same active request when the data and parameters are identical:

```text
Component A ─┐
Component B ─┼→ shared request → API
Component C ─┘
```

This reduces:

* network usage
* server load
* duplicate processing
* inconsistent intermediate states

---

## When to use request deduplication

Use request deduplication when:

* multiple components need the same resource
* several hooks request the same endpoint
* navigation causes repeated requests
* multiple parts of a dashboard use the same data
* requests can be triggered concurrently
* the application uses a shared server-state layer

---

## When not to create custom deduplication

Do not create a custom request registry for a small application if the problem does not exist.

For:

```text
one page
→ one request
→ one component
```

basic request handling may be enough.

Do not add infrastructure simply to say that the application has "request deduplication."

---

## Request identity

Deduplication requires a reliable definition of what constitutes the same request.

These are different:

```text
GET /api/drivers?page=1
GET /api/drivers?page=2
```

These are also different:

```text
GET /api/drivers/123
GET /api/drivers/456
```

Request identity should include all relevant parameters.

For example:

```text
["drivers", { page: 1, status: "REGISTERED" }]
```

and:

```text
["drivers", { page: 2, status: "REGISTERED" }]
```

must not be treated as the same resource.

---

## Stable query keys

When using a query or server-state library, use stable query keys.

Examples:

```text
["drivers"]
["drivers", driverId]
["drivers", { page, status }]
["violations", { driverId, year }]
```

A query key should represent the inputs that determine the returned data.

Do not omit meaningful parameters from the key.

---

# Request Cancellation

## Purpose

Cancel requests that are no longer relevant.

This is especially useful for:

* search
* autocomplete
* changing filters
* changing routes
* changing selected IDs
* rapidly changing form values
* abandoned pages
* expensive API requests

---

## Search example

Without cancellation:

```text
User types:
d
dr
dri
driv
driver

Requests:
d
dr
dri
driv
driver
```

An older request can potentially finish after a newer request.

This can cause:

```text
new result
↓
old result arrives
↓
old result overwrites new result
```

Use cancellation or request identity to prevent obsolete results from replacing current results.

---

## AbortController

When using the browser Fetch API, use `AbortController` where appropriate.

The request lifecycle should account for:

```text
start
↓
active
↓
completed
```

or:

```text
start
↓
aborted
```

Intentional cancellation should not normally be treated as a user-visible server failure.

---

## Cleanup

Cancel or clean up requests when:

* the component no longer needs the result
* the user navigates away
* the search query changes
* the selected resource changes
* the request becomes obsolete

Do not allow abandoned requests to continue indefinitely when they provide no remaining value.

---

# Race Conditions

## Problem

Consider:

```text
Request A starts
↓
Request B starts
↓
Request B finishes
↓
UI shows B
↓
Request A finishes
↓
UI accidentally shows A
```

This is a race condition.

The frontend must ensure that older, irrelevant responses do not overwrite newer state.

---

## Solutions

Depending on the architecture, use:

* request cancellation
* stable request identity
* query libraries
* sequence numbers
* active-request checks
* server-state management

Do not solve race conditions by simply adding arbitrary delays.

---

# Debouncing

## Purpose

Debouncing prevents high-frequency events from triggering excessive requests.

Good examples:

* search fields
* autocomplete
* remote filtering
* username availability
* address lookup
* server-side validation

Example:

```text
d
dr
dri
driv
drive
driver
```

Instead of sending six requests, wait until the user pauses:

```text
user stops typing
↓
short delay
↓
request "driver"
```

---

## Do not debounce everything

Do not debounce:

* normal button clicks
* navigation
* form submission
* destructive actions
* interactions where immediate response is expected

Debouncing is primarily for **high-frequency events**.

---

## Debounce timing

The delay should depend on the interaction.

A search field may tolerate a short delay.

A validation interaction may require a different strategy.

Do not blindly use one global debounce value for every feature.

The goal is:

```text
reduce unnecessary requests
```

without making the interface feel sluggish.

---

# Throttling

## Purpose

Throttling limits how frequently an event can trigger an operation.

Useful for:

* scroll events
* resize events
* pointer movement
* continuous UI measurement
* analytics
* high-frequency browser events

Example:

```text
scroll event
scroll event
scroll event
scroll event
scroll event
```

should not necessarily trigger expensive work for every event.

---

## Debounce vs throttle

### Debounce

Wait until activity stops.

Best for:

```text
search input
autocomplete
validation
```

### Throttle

Allow execution at a controlled frequency while activity continues.

Best for:

```text
scroll
resize
pointer movement
```

Choose based on the behavior required.

---

# Caching

## Purpose

Caching stores previously retrieved server data so it can potentially be reused.

Benefits can include:

* faster navigation
* fewer requests
* lower network usage
* lower server load
* better perceived performance
* smoother transitions

---

## Do not cache everything

Consider whether the data is:

* static
* slowly changing
* frequently changing
* user-specific
* sensitive
* expensive to retrieve
* large
* temporary

Caching strategy should depend on the data.

---

## Cache lifetime

Determine how long data can reasonably remain available.

Examples:

```text
Static reference data
→ long cache

Frequently changing dashboard
→ short freshness period

User profile
→ moderate freshness

Highly sensitive data
→ careful caching or no persistent caching
```

Do not choose cache durations arbitrarily.

---

# Stale-While-Revalidate

## Purpose

Stale-while-revalidate allows the UI to display existing cached data immediately while requesting a fresh version in the background.

Flow:

```text
cached data exists
↓
show cached data
↓
request fresh data
↓
receive response
↓
update cache
↓
update UI
```

This can make navigation and repeated views feel significantly faster.

---

## When to use SWR

Good candidates include:

* dashboards
* lists
* profiles
* reference information
* frequently visited pages
* data that changes periodically
* data where slight staleness is acceptable

---

## Background refresh

Do not replace existing useful data with a full-screen loading state during background refresh.

Prefer:

```text
Existing data
+
subtle "Updating..." state
```

over:

```text
Existing data disappears
↓
LOADING...
↓
Existing data returns
```

unless the existing data is no longer safe or valid to display.

---

## Initial loading vs refreshing

Distinguish:

### Initial loading

```text
No data
↓
Loading
↓
Data
```

from:

### Background refresh

```text
Existing data
↓
Refreshing
↓
Updated data
```

These are different UX states.

---

# Stale Data Safety

Do not use SWR when stale data can cause harmful decisions.

Use caution with:

* permissions
* authorization state
* financial information
* critical transaction status
* time-sensitive availability
* security-sensitive information

If stale information is dangerous, prioritize correctness over perceived speed.

---

# Cache Invalidation

## Purpose

When a mutation changes server data, cached information may become stale.

Example:

```text
PATCH /api/drivers/123
```

may affect:

```text
["drivers", "123"]
```

and:

```text
["drivers"]
```

if the list contains the modified driver.

---

## Targeted invalidation

Prefer invalidating affected resources rather than clearing everything.

Avoid:

```text
mutation
↓
clear entire cache
↓
refetch entire application
```

Prefer:

```text
mutation
↓
identify affected data
↓
invalidate affected cache
↓
refresh only necessary data
```

---

# Mutation Responses

If a mutation returns the updated resource, use that response when appropriate.

Example:

```text
PATCH /drivers/123
↓
server returns updated driver
↓
update cached driver
```

This may avoid an unnecessary:

```text
GET /drivers/123
```

immediately afterward.

The server response remains authoritative.

---

# Optimistic Updates

## Purpose

Optimistic updates make the interface respond immediately before the server confirms the mutation.

Example:

```text
User clicks "Complete"
↓
UI immediately shows "Completed"
↓
request sent
↓
server confirms
```

This can make interactions feel significantly faster.

---

## Good candidates

Optimistic updates are appropriate when:

* success is highly likely
* the expected result is predictable
* rollback is simple
* immediate feedback matters
* the action is reversible
* temporary inconsistency is acceptable

Examples:

* favorite toggles
* likes
* simple status changes
* preference switches
* lightweight reordering

---

## Poor candidates

Avoid optimistic updates when:

* server validation is complex
* failure is common
* rollback is difficult
* the operation is irreversible
* financial consequences exist
* server-generated values determine the result
* permissions may change the result
* correctness is more important than immediate feedback

Do not optimistically pretend that a critical operation succeeded.

---

# Optimistic Update Lifecycle

Use a predictable lifecycle:

```text
capture previous state
↓
apply optimistic state
↓
send request
↓
success → reconcile with server
↓
failure → rollback
```

Do not implement optimistic UI without a failure strategy.

---

# Rollback

Before changing cached or visible state optimistically, preserve enough information to restore the previous state.

Example:

```text
Before:
status = PENDING

Optimistic:
status = APPROVED

Request fails:

Rollback:
status = PENDING
```

Rollback must restore the correct state.

Do not simply refetch blindly if that can create unnecessary requests or race conditions.

---

# Concurrent Optimistic Updates

Be careful when multiple mutations affect the same resource.

Example:

```text
Mutation A
↓
Mutation B
↓
B succeeds
↓
A fails
```

A simplistic rollback of A could accidentally undo B.

For complex concurrent mutations, use a state-management approach that can reconcile updates safely.

Do not rely on naive snapshots when mutation concurrency becomes complicated.

---

# Server Reconciliation

The server is authoritative.

Even if an optimistic update appears correct, the server may return:

* normalized values
* generated IDs
* timestamps
* calculated totals
* transformed fields
* updated permissions
* server-side status
* additional metadata

Always reconcile optimistic state with the confirmed server result when appropriate.

---

# Retry with Backoff

## Purpose

Retry requests when a failure may be temporary.

Potential candidates:

* temporary network interruption
* timeout
* transient server failure
* temporary infrastructure issue
* recoverable rate limiting

Do not retry every error.

---

# Permanent vs transient errors

Generally do not automatically retry:

* invalid credentials
* forbidden requests
* invalid parameters
* validation failures
* malformed requests
* known not-found responses
* permanent business-rule failures

Retrying these does not solve the underlying problem.

---

# Exponential Backoff

Avoid:

```text
failure
↓
retry immediately
↓
failure
↓
retry immediately
↓
failure
↓
retry immediately
```

Prefer progressively increasing delays:

```text
attempt 1
↓
short delay

attempt 2
↓
longer delay

attempt 3
↓
longer delay
```

This reduces pressure on an already failing service.

---

# Jitter

When many clients retry simultaneously, they can accidentally synchronize.

Example:

```text
1000 clients
↓
all retry after exactly 2 seconds
↓
server receives another traffic spike
```

Use jitter where appropriate to introduce small random variations to retry delays.

Conceptually:

```text
backoff delay
+
random variation
```

---

# Retry Limits

Never retry forever.

Define:

* maximum attempts
* maximum retry duration
* stopping conditions

Eventually the request should enter a visible failure state.

---

# Retry UX

For user-visible actions, do not silently retry indefinitely.

Useful states include:

```text
Saving...
```

then:

```text
Connection interrupted.
Retrying...
```

then:

```text
Saved
```

or:

```text
Unable to save.
Try again.
```

Do not expose unnecessary technical details.

---

# Retrying Mutations

Be especially careful with mutation retries.

Consider:

```text
POST /api/transactions
```

The server may successfully process the request even if the client never receives the response.

Blindly retrying could create duplicate operations.

For important non-idempotent operations, consider:

* idempotency keys
* server-side duplicate detection
* operation IDs
* checking operation status
* appropriate HTTP semantics

Do not assume that retrying a mutation is always safe.

---

# Smart Polling

## Purpose

Polling periodically requests updated data.

It is useful when:

* data changes periodically
* real-time communication is unnecessary
* the backend does not provide a stream
* a delay of a few seconds is acceptable

Examples:

* job status
* export progress
* import progress
* dashboard metrics
* processing status
* device status

---

# Do not poll by default

Avoid:

```text
setInterval(fetchData, 1000)
```

unless the product genuinely requires that frequency.

Ask:

* How fresh must the data be?
* How frequently does it actually change?
* How expensive is the request?
* How many users may poll simultaneously?
* Is the page visible?
* Can the server push updates instead?
* Does the user actually benefit from second-by-second updates?

---

# Adaptive polling

Polling frequency can change based on application state.

Example:

```text
job running
→ poll frequently

job almost finished
→ continue polling

job completed
→ stop polling
```

For long-running jobs, adaptive polling can reduce unnecessary traffic.

---

# Visibility-aware polling

When the user cannot see the page, continuous polling may provide little value.

Consider:

```text
page visible
→ normal polling

page hidden
→ slow down or pause

page visible again
→ refresh immediately
→ resume normal polling
```

This reduces unnecessary network usage.

---

# Focus-aware refresh

When a user returns to a page, refresh data if it may have become stale.

This can sometimes replace aggressive continuous polling.

Example:

```text
User leaves dashboard
↓
dashboard is not continuously refreshed
↓
user returns
↓
fetch current data
```

---

# Polling backoff

If polling requests repeatedly fail, do not continue at the normal frequency.

Instead:

```text
normal polling
↓
failure
↓
slower polling
↓
continued failure
↓
even slower polling
```

Restore normal polling after successful responses where appropriate.

---

# Polling stop conditions

Every polling system must have a reason to stop.

Examples:

* job completed
* job failed
* user leaves page
* component unmounts
* operation cancelled
* maximum duration reached
* server reports completion
* data no longer matters

Never create an accidental infinite polling loop.

---

# Streaming UI

## Purpose

Streaming allows users to receive partial results progressively instead of waiting for the complete response.

Good candidates:

* AI-generated content
* long reports
* document generation
* long-running processing
* progressive results
* live event feeds

---

# Do not stream everything

Do not use streaming for:

* small JSON responses
* simple CRUD
* ordinary form submissions
* tiny configuration data
* responses that are already fast

Streaming adds implementation and error-handling complexity.

Use it when partial results actually improve the user experience.

---

# Streaming UX

The user should understand the state:

```text
Starting
↓
Generating
↓
Partial result
↓
More result
↓
Complete
```

Avoid making streaming look like a frozen page.

---

# Streaming cancellation

For long-running streams, allow cancellation when appropriate.

Example:

```text
Generate
↓
Generating...
↓
Stop
```

Stopping should:

* cancel the request
* close the stream
* clean up resources
* leave useful received content intact when appropriate

---

# Streaming failure

A stream may fail after partial content has already arrived.

Do not automatically erase everything.

Prefer:

```text
Partial result
+
"Generation interrupted"
+
Retry / Continue action
```

when appropriate.

---

# Streaming cleanup

Clean up streams when:

* component unmounts
* navigation occurs
* operation is cancelled
* stream completes
* stream fails

Do not leave connections open after they are no longer needed.

---

# Server-Sent Events

SSE is appropriate when:

* updates primarily travel from server to client
* the browser needs ongoing updates
* bidirectional communication is unnecessary

Potential examples:

* notifications
* status updates
* live progress
* server events

Do not use WebSockets simply because they sound more advanced.

---

# WebSockets

Use WebSockets when genuine bidirectional real-time communication is required.

Examples:

* chat
* multiplayer systems
* collaborative editing
* live control systems

Do not introduce WebSockets for a normal dashboard that only needs occasional updates.

---

# Streaming vs Polling

Use the simplest appropriate mechanism.

```text
Rarely changing data
→ normal request

Periodically changing data
→ polling

Server-to-client live updates
→ SSE

Bidirectional real-time communication
→ WebSocket
```

Do not combine these mechanisms without a clear reason.

---

# Pagination

## Purpose

Pagination prevents large datasets from being loaded and rendered unnecessarily.

Instead of:

```text
GET /drivers
→ 10,000 records
```

prefer something such as:

```text
GET /drivers?page=1&limit=25
```

when the backend supports it.

---

# When to use pagination

Use pagination for:

* administrative tables
* transaction records
* driver records
* violation history
* user lists
* large search results
* audit logs

---

# Pagination UX

Provide appropriate:

* current page
* page size
* next/previous controls
* loading state
* empty state
* error state

Do not unnecessarily reset pagination after unrelated actions.

---

# Cursor pagination

Cursor-based pagination can be useful for:

* large datasets
* continuously changing datasets
* feeds
* activity streams

Do not choose cursor pagination purely because it sounds more advanced.

Coordinate the frontend with the backend API design.

---

# Infinite scrolling

Infinite scrolling can be useful for:

* feeds
* media
* discovery
* continuous browsing

Avoid it when users need:

* exact page numbers
* predictable navigation
* easy comparison
* a stable location
* administrative table workflows

---

# Prevent duplicate page loads

When loading the next page:

```text
page 2 request active
```

do not allow another interaction to accidentally trigger:

```text
page 2 request
```

again.

Track active pagination requests or use a server-state library that handles this.

---

# Pagination cache

Do not overwrite page 1 when page 2 loads.

Treat pages as separate pieces of data while allowing them to be combined for display.

Example:

```text
page 1
+
page 2
+
page 3
```

can become:

```text
combined visible list
```

without losing the individual request identity.

---

# Prefetching

## Purpose

Prefetch data when there is a strong likelihood that the user will need it soon.

Examples:

* next pagination page
* predictable next route
* selected item's detail
* common dashboard navigation

---

# Good prefetching

Example:

```text
User is viewing page 1
↓
Page 2 is likely next
↓
Prefetch page 2
↓
User clicks Next
↓
Page 2 appears quickly
```

---

# Bad prefetching

Avoid:

```text
Load every possible page
Load every possible driver
Load every possible modal
Load every route
Load every image
```

This simply moves the performance problem somewhere else.

Prefetch should be intentional.

---

# Prefetching and network conditions

Consider whether the user is on:

* slow network
* limited mobile data
* unstable connection

Do not aggressively prefetch large resources when the cost outweighs the benefit.

---

# Cache + Prefetch

Prefetched data should normally enter the same server-state/cache strategy used by the application.

Avoid creating a separate hidden data store solely for prefetched content.

---

# Loading States

Every asynchronous operation should consider its state.

At minimum:

```text
idle
loading
success
empty
error
```

Depending on the application, also consider:

```text
refreshing
saving
retrying
cancelled
offline
partial
streaming
```

---

# Initial Loading

When no data exists yet, provide an appropriate loading state.

Examples:

* skeleton
* spinner
* progress indicator
* inline loading state

Choose based on the component.

Do not use a full-screen spinner for every small request.

---

# Background Loading

If useful data already exists, avoid hiding it while refreshing.

Prefer:

```text
Existing content
+
small refresh indicator
```

rather than:

```text
Existing content removed
+
large spinner
```

---

# Skeleton Loading

Use skeletons when they improve perceived continuity.

A good skeleton should roughly match the structure of the eventual content.

Avoid creating complicated animated skeleton systems for tiny components.

---

# Error States

Every request should have a failure path.

Do not leave:

```text
Loading...
```

on the screen forever.

Provide an appropriate recovery path when possible:

```text
Failed to load drivers.
[Try again]
```

---

# Empty vs Error

Do not confuse these states.

### Empty

```text
Request succeeded.
There are no records.
```

### Error

```text
Request failed.
The records are unknown.
```

These should have different UI and messaging.

---

# Retry UI

For recoverable errors, provide a retry action where appropriate.

Example:

```text
Unable to load transactions.

[Try again]
```

Do not force the user to reload the entire page.

---

# Loading Buttons

For mutations:

```text
Save
```

can become:

```text
Saving...
```

and after success:

```text
Saved
```

Prevent duplicate submission while the operation is active when appropriate.

---

# Mutation Feedback

Every important mutation should communicate its outcome.

Possible states:

```text
idle
↓
saving
↓
success
```

or:

```text
idle
↓
saving
↓
error
```

Do not make users guess whether the action worked.

---

# Preventing Duplicate Mutations

Double-clicking should not accidentally trigger:

```text
POST
POST
```

when only one operation was intended.

Use appropriate mechanisms such as:

* temporary button disable
* mutation state
* idempotency keys
* request deduplication where appropriate

For high-risk operations, backend idempotency is more important than simply disabling a button.

---

# Sequential and Parallel Requests

Independent requests should run in parallel when safe.

Instead of:

```text
request drivers
↓
request enforcers
↓
request violations
```

if there are no dependencies, consider:

```text
drivers ─┐
enforcers ├→ parallel
violations┘
```

This can reduce total loading time.

---

# Dependent Requests

Requests should be sequential when one genuinely depends on another.

Example:

```text
authenticate
↓
obtain user identity
↓
load user-specific data
```

Do not artificially serialize independent requests.

---

# Waterfall Prevention

Avoid unnecessary request waterfalls.

Example of a problematic flow:

```text
page loads
↓
request user
↓
request permissions
↓
request dashboard
↓
request statistics
```

If permissions and dashboard data can safely be loaded concurrently, do so.

---

# Request Batching

When multiple requests can be safely combined and the backend supports it, batching can reduce network overhead.

Example:

```text
GET driver 1
GET driver 2
GET driver 3
```

may become:

```text
GET drivers?ids=1,2,3
```

when appropriate.

Do not create complicated batching logic unless the number of requests justifies it.

---

# API Payload Efficiency

Performance is not only about request count.

Also consider:

* response size
* unnecessary fields
* image size
* pagination
* compression
* serialization
* repeated data

Do not request data the UI does not need when the API supports selective fields.

---

# Image and Media Requests

Large images can dominate network performance.

Use:

* appropriate dimensions
* responsive images
* lazy loading
* compression
* modern formats where appropriate
* placeholders when useful

Do not load full-resolution assets when a thumbnail is sufficient.

---

# Lazy Loading

Lazy-load expensive resources when they are not immediately required.

Potential candidates:

* large images
* heavy components
* maps
* charts
* editor libraries
* rarely visited routes

Do not lazy-load tiny elements simply because lazy loading exists.

---

# Code Splitting

Load large application features only when necessary when the project benefits from it.

Good candidates:

* admin modules
* charting libraries
* editors
* maps
* large feature-specific components

Do not split every component into a separate chunk.

---

# React Considerations

When working with React, be careful about component lifecycle and request execution.

Watch for:

* `useEffect`
* dependency arrays
* changing object references
* route transitions
* remounts
* state updates triggering effects
* duplicate development behavior

Do not accidentally create:

```text
render
↓
request
↓
state update
↓
render
↓
request
↓
state update
```

---

# Effect Dependencies

If a request depends on:

* `driverId`
* `search`
* `page`
* `status`
* `filter`

make sure the request lifecycle correctly responds to changes.

Do not remove dependencies merely to stop repeated requests.

Fix the request architecture instead.

---

# Server-State Libraries

When the project already uses a server-state library, follow its established patterns.

A suitable library may provide:

* caching
* deduplication
* retries
* invalidation
* mutations
* optimistic updates
* stale times
* background refetching

Do not build a second custom caching system alongside an existing server-state library without a strong reason.

---

# Library Selection

Do not automatically add a server-state library to every project.

For a small application:

```text
fetch
+
local state
```

may be enough.

For a larger application with:

* many endpoints
* shared server data
* complex invalidation
* mutations
* background refresh
* pagination
* optimistic updates

a dedicated server-state solution may provide substantial value.

Choose based on actual complexity.

---

# Custom API Client

Centralize genuinely shared behavior where appropriate.

Potential shared responsibilities:

* base URL
* authentication headers
* request serialization
* response parsing
* error normalization
* timeout handling

Avoid making one enormous API abstraction that hides endpoint-specific behavior.

---

# Authentication and Server State

Authentication state should be handled separately from ordinary cached application data.

Do not assume that:

```text
cached data exists
```

means:

```text
user is authorized to access it
```

The backend must continue enforcing authorization.

---

# Authentication Expiration

When authentication expires:

* stop treating the request as an ordinary server error
* avoid infinite retries
* handle session expiration centrally where appropriate
* prevent multiple components from independently refreshing authentication
* redirect or re-authenticate according to the application's auth design

Do not let:

```text
401
↓
retry
↓
401
↓
retry
```

continue indefinitely.

---

# Cache Security

Be careful when caching:

* personal information
* administrative data
* financial data
* private documents
* authentication-related information
* privileged information

Consider:

* cache lifetime
* cache scope
* logout behavior
* account switching
* sensitive-data invalidation

---

# User Switching

If the application supports:

```text
User A logout
↓
User B login
```

ensure that User A's cached private data does not appear to User B.

User-specific cache keys or cache clearing may be required.

This is both a correctness and security concern.

---

# Logout Cleanup

On logout, consider cleaning up:

* user-specific cache
* active requests
* polling
* streams
* subscriptions
* optimistic state
* sensitive local state

Do not allow private data to remain visible after the session ends.

---

# Mobile Applications

For React Native and mobile projects, consider:

* bandwidth
* battery usage
* intermittent connectivity
* background execution limits
* slower devices
* limited storage
* mobile data usage

Aggressive polling is especially expensive on mobile.

Prefer:

```text
targeted refresh
+
caching
+
efficient requests
```

when appropriate.

---

# Offline Behavior

Do not automatically build a complete offline-first architecture.

Offline-first can require:

* local persistence
* mutation queues
* synchronization
* conflict resolution
* retry
* reconciliation

Implement it only when offline usage is an actual project requirement.

---

# Offline-Aware UX

If the application supports offline behavior, distinguish:

```text
online
offline
syncing
synced
sync failed
```

Do not pretend that a mutation has reached the server when it has only been queued locally.

---

# Conflict Resolution

If local and server data can diverge, define what happens.

Possible strategies include:

* server wins
* client wins
* latest update wins
* field-level merge
* manual conflict resolution

Do not silently overwrite meaningful user changes.

---

# Cache Consistency

When data changes, consider where that data appears.

For example, updating a driver could affect:

```text
Driver detail
Driver list
Dashboard statistics
Search results
Violation records
Vehicle assignments
```

Do not update only the currently visible component if other cached representations become stale.

---

# Avoid Full Application Refetches

Avoid:

```text
small mutation
↓
reload everything
```

when targeted invalidation is possible.

A small change should ideally result in a small amount of synchronization work.

---

# Data Freshness

Freshness should be based on actual requirements.

Example:

```text
Static reference data
→ long freshness

Normal profile
→ moderate freshness

Administrative dashboard
→ shorter freshness

Live operational status
→ very short refresh or streaming
```

Do not use one universal refresh policy.

---

# Freshness vs Performance

There is always a tradeoff:

```text
more freshness
↔
more requests
```

and:

```text
more caching
↔
potentially older data
```

Choose based on what the user actually needs.

Do not make everything real-time.

---

# Perceived Performance

Improve perceived performance using:

* cached data
* SWR
* optimistic updates
* skeletons
* progressive rendering
* streaming
* immediate button feedback
* inline refresh indicators
* preserving existing content during refresh

Do not fake progress.

Do not show "100%" before an operation is actually complete.

---

# Stable Layouts

Loading states should avoid unnecessary layout shifts.

Prefer skeletons or reserved space when the final content has predictable dimensions.

Avoid:

```text
empty space
↓
large content suddenly appears
↓
everything moves
```

when a stable layout can be maintained.

---

# Animation and Data Loading

Data loading should integrate with the project's visual design system.

Use subtle motion for:

* appearing content
* refreshing indicators
* state changes
* success feedback
* streaming content

Do not animate every incoming data item aggressively.

Performance feedback should feel polished rather than theatrical.

---

# Reduced Motion

Respect:

```css
@media (prefers-reduced-motion: reduce)
```

where appropriate.

Data-loading animations should not become a usability problem for users who prefer reduced motion.

---

# Request Timeout

Long-running requests should have appropriate timeout behavior where the platform and operation allow it.

Different operations may need different expectations.

For example:

```text
Autocomplete
→ short timeout

Normal CRUD
→ moderate timeout

Large report generation
→ longer operation / background job
```

Do not use one arbitrary timeout for every request.

---

# Error Classification

When useful, distinguish:

* network error
* timeout
* authentication error
* authorization error
* validation error
* not found
* rate limiting
* server error
* cancellation

Different errors can require different behavior.

For example:

```text
401
→ authentication handling

422
→ validation feedback

429
→ backoff

500
→ possible retry

Abort
→ usually no error message
```

Do not treat every failure as:

```text
Something went wrong.
```

internally.

---

# Rate Limiting

Respect backend rate limits.

If the server communicates that requests should be delayed:

* slow down
* respect retry information
* avoid aggressive retries
* avoid creating request loops

Do not respond to rate limiting by increasing request frequency.

---

# Retry and Rate Limiting

A rate-limited response is a signal to reduce pressure.

Do not use:

```text
429
↓
retry immediately
↓
429
↓
retry immediately
```

Prefer controlled backoff.

If the server provides a retry delay, respect it where appropriate.

---

# Request Queuing

Use request queues only when the application genuinely requires controlled synchronization.

Potential cases:

* offline mutation synchronization
* serialized device operations
* constrained background actions
* operations that must execute in order

Do not create a global queue for ordinary API calls without a clear requirement.

---

# Sequential Mutations

If mutation B depends on mutation A succeeding, execute them in the appropriate order.

Example:

```text
create driver
↓
receive driver ID
↓
upload driver document
```

Do not execute dependent operations in parallel.

---

# Parallel Mutations

If operations are independent and safe to execute together:

```text
upload photo
upload document
load metadata
```

may potentially run in parallel.

Use judgment based on:

* backend capacity
* dependencies
* user experience
* failure handling

---

# Partial Failure

When multiple requests execute together, one may fail while others succeed.

Do not assume:

```text
one failure
=
everything failed
```

Represent partial success when appropriate.

Example:

```text
Driver updated successfully.
Vehicle document upload failed.

[Retry upload]
```

This is more useful than:

```text
Operation failed.
```

---

# Batch Operations

For actions affecting many records, consider whether the backend supports batch operations.

Instead of:

```text
PATCH item 1
PATCH item 2
PATCH item 3
PATCH item 4
```

a suitable API may support:

```text
PATCH /items/batch
```

Do not create frontend batching that the backend cannot safely support.

---

# Search Performance

Search interfaces should prioritize:

* fast feedback
* debounce
* cancellation
* stable results
* empty-state clarity
* loading feedback
* error recovery

Avoid sending requests for every keystroke unless the application specifically requires it.

---

# Search Result Race Prevention

For search:

```text
search "dr"
↓
request A

search "driver"
↓
request B

B completes
↓
show driver results

A completes later
↓
DO NOT replace driver results with "dr" results
```

Use cancellation or request identity.

---

# Search Caching

Short-lived caching can be useful when users repeatedly return to the same query.

Example:

```text
"driver"
↓
results cached

user searches something else
↓
returns to "driver"
↓
cached results available
```

Do not keep unlimited search queries forever.

---

# Dashboard Data

Dashboards often contain multiple data sources.

Do not make the entire dashboard depend on one giant request unless the backend architecture requires it.

Consider independent sections when appropriate:

```text
Summary
↓
independent data

Recent transactions
↓
independent data

Driver statistics
↓
independent data
```

This allows one section to load or fail without necessarily blocking everything.

---

# Dashboard Polling

Only poll dashboard data that actually changes.

Do not refresh static configuration every five seconds simply because another metric is live.

Example:

```text
Live active drivers
→ polling

Historical yearly statistics
→ normal cache

Static system information
→ no polling
```

---

# Polling Multiple Resources

Avoid creating many independent intervals when one coordinated refresh is sufficient.

Bad:

```text
interval A → statistics
interval B → drivers
interval C → notifications
interval D → transactions
```

when all need the same refresh cadence.

Consider a more controlled synchronization strategy.

---

# Smart Refresh

Use the user's context to determine whether refresh is useful.

Examples:

```text
user viewing dashboard
→ refresh

user editing a form
→ do not aggressively replace form data

user viewing static page
→ no refresh

user returns to page
→ refresh if stale
```

Never let background synchronization unexpectedly overwrite active user input.

---

# Protect User Input

This is critical.

If the user is editing:

```text
driver information
```

and a background refresh occurs, do not blindly replace the form values with server data.

Separate:

```text
server state
```

from:

```text
unsaved local edits
```

and reconcile intentionally.

---

# Forms and Server State

Forms should generally maintain their own temporary editing state.

Do not continuously synchronize every server refresh directly into an active form unless the UX explicitly requires collaborative/live editing.

---

# Long-Running Jobs

For operations such as:

* report generation
* export
* import
* document processing
* media processing

consider representing the process as a job:

```text
start job
↓
job ID
↓
track progress
↓
completed
```

The tracking mechanism may use:

* polling
* SSE
* WebSocket
* streaming

depending on the backend.

---

# Job Polling

For a background job:

```text
queued
↓
processing
↓
processing
↓
completed
```

poll only while the job is active.

Stop once:

```text
completed
```

or:

```text
failed
```

is reached.

---

# Progress Accuracy

Never fabricate progress values.

If the backend only knows:

```text
processing
```

do not display:

```text
73%
```

unless that percentage has a meaningful basis.

Use:

```text
Processing...
```

when exact progress is unavailable.

---

# Streaming and Progress

Streaming partial output and progress indicators solve different problems.

Use streaming when:

```text
partial result itself is useful
```

Use progress when:

```text
the operation has measurable progress
```

They may be combined when appropriate.

---

# Large Lists

For large lists, consider:

* pagination
* infinite loading
* virtualization
* filtering
* server-side search
* selective fields

Do not render thousands of complex components if only a small subset is visible.

---

# Virtualization

Virtualization renders only the portion of a large list currently needed.

Use it when the dataset or component complexity creates a measurable rendering problem.

Do not add virtualization to:

```text
10-item list
```

just because the technique exists.

---

# Performance Measurement

Before significant optimization, identify the problem.

Useful measurements may include:

* request duration
* number of requests
* duplicate request count
* payload size
* cache hit rate
* retry frequency
* render time
* interaction latency
* time to useful content
* list rendering performance

Optimize based on evidence where possible.

---

# Do Not Optimize Vanity Metrics

Do not sacrifice usability simply to improve an isolated metric.

For example:

```text
fewer API requests
```

is not automatically better if it means:

```text
stale information
```

Similarly:

```text
smaller bundle
```

is not automatically better if it creates poor navigation or excessive loading boundaries.

Optimize the product experience, not numbers in isolation.

---

# Complexity Budget

Every advanced pattern introduces complexity.

Consider:

```text
benefit
vs
implementation complexity
vs
maintenance cost
```

Before adding a pattern, ask:

> Will this complexity solve a real problem?

If not, do not add it.

---

# Combining Patterns

Patterns can complement each other.

Reasonable combinations include:

```text
Caching
+
Request deduplication
```

```text
SWR
+
Background refresh
```

```text
Optimistic update
+
Cache reconciliation
```

```text
Retry
+
Exponential backoff
+
Jitter
```

```text
Polling
+
Visibility awareness
+
Backoff
```

```text
Search
+
Debounce
+
Cancellation
+
Short-lived cache
```

```text
Pagination
+
Prefetching
+
Caching
```

These combinations should still be justified by actual requirements.

---

# Avoid Excessive Pattern Stacking

Do not automatically build:

```text
SWR
+
polling
+
streaming
+
prefetching
+
optimistic updates
+
offline queue
+
multiple cache layers
+
custom request deduplication
```

into a simple CRUD screen.

Complexity should correspond to actual application complexity.

---

# Pattern Decision Matrix

| Situation                             | Recommended pattern     |
| ------------------------------------- | ----------------------- |
| Static content                        | None                    |
| One-time API request                  | Basic request lifecycle |
| Same request triggered concurrently   | Request deduplication   |
| Frequently revisited data             | Cache                   |
| Slightly stale data is acceptable     | SWR                     |
| User typing into search               | Debounce                |
| Obsolete search requests              | Cancellation            |
| High-frequency browser events         | Throttle                |
| Temporary network failure             | Retry with backoff      |
| Large dataset                         | Pagination              |
| Continuous browsing                   | Infinite loading        |
| Extremely large rendered list         | Virtualization          |
| Predictable next resource             | Prefetch                |
| Periodically changing data            | Polling                 |
| Frequently changing visible data      | Smart polling           |
| Server pushes one-way updates         | SSE                     |
| Bidirectional real-time communication | WebSocket               |
| Progressive generated content         | Streaming               |
| Predictable reversible mutation       | Optimistic update       |
| Offline requirement                   | Offline synchronization |
| No actual performance problem         | Keep it simple          |

---

# Anti-Patterns

## 1. Fetching on every render

Avoid request logic that accidentally executes whenever a component renders.

```text
render
↓
fetch
↓
state update
↓
render
↓
fetch
```

This can create request loops.

---

## 2. Refetching everything after every mutation

Avoid:

```text
save one driver
↓
reload entire application
```

Prefer targeted synchronization.

---

## 3. Full-screen loading for every refresh

Avoid:

```text
data exists
↓
refresh
↓
hide everything
↓
spinner
↓
show same data again
```

Prefer preserving useful content during background refresh.

---

## 4. Polling every second

Do not use one-second polling as a default.

Determine actual freshness requirements first.

---

## 5. Infinite polling

Every poll needs a stop condition.

---

## 6. Retry forever

Every retry system needs:

* maximum attempts
* maximum duration
* failure state

---

## 7. Retrying validation errors

Do not retry errors that require user correction.

---

## 8. Retrying mutations blindly

A mutation may have succeeded even when the response was lost.

Consider idempotency before automatic retries.

---

## 9. Optimistically updating critical operations

Do not use optimistic updates simply because they look faster.

Correctness matters more for critical operations.

---

## 10. Caching private data without lifecycle management

Do not allow private data to survive:

```text
logout
↓
new user login
```

without appropriate isolation.

---

## 11. Aggressive prefetching

Do not download everything users might possibly click.

---

## 12. Debouncing ordinary actions

Do not add artificial delays to normal button interactions.

---

## 13. Streaming tiny responses

Streaming is not automatically better.

---

## 14. WebSockets everywhere

Use WebSockets only when the application actually needs bidirectional real-time communication.

---

## 15. Multiple sources of truth

Avoid:

```text
server cache
+
component state
+
global state
+
local storage
```

all independently representing the same server resource without a clear synchronization strategy.

---

# Security Considerations

Performance optimizations must never weaken security.

Never:

* use cached data as proof of authorization
* skip backend authorization
* expose another user's cached data
* retain sensitive data indefinitely
* trust optimistic state as confirmed server state
* treat stale permissions as authoritative
* bypass authentication because data was previously fetched

The frontend can improve UX.

The backend remains responsible for security enforcement.

---

# Data Privacy

When data is sensitive, consider:

* whether it should be cached
* where it is cached
* how long it remains cached
* whether it survives logout
* whether multiple users can use the same device
* whether persistent storage is appropriate

Do not persist sensitive information merely for convenience.

---

# Mobile Network Efficiency

For React Native applications, especially applications intended for mobile users:

Prioritize:

* small payloads
* caching
* deduplication
* pagination
* controlled polling
* request cancellation
* efficient images
* minimal unnecessary refreshes

Avoid wasting mobile data on requests the user does not need.

---

# Battery Considerations

Continuous polling, location updates, streams, and background synchronization can consume battery.

Use aggressive refresh only when the product genuinely requires it.

For mobile:

```text
fresh enough
```

is often better than:

```text
as fresh as technically possible
```

---

# Accessibility

Asynchronous behavior must remain understandable without relying exclusively on animation.

Consider:

* accessible loading messages
* focus management
* error announcements
* keyboard interaction
* clear disabled states
* screen-reader feedback
* reduced motion

Do not communicate success only through color or animation.

---

# Reduced Motion

Respect:

```css
@media (prefers-reduced-motion: reduce)
```

when implementing loading, streaming, refresh, or success animations.

The data behavior should remain understandable without motion.

---

# UX Consistency

All asynchronous interactions should feel like they belong to the same application.

For example:

```text
Button loading
Table loading
Card loading
Modal loading
```

should use a consistent visual language.

Do not invent a different loading animation for every component.

---

# Data Performance and Visual Design

Performance behavior should support the project's visual design.

Use:

* subtle refresh indicators
* clean skeletons
* meaningful progress states
* smooth content transitions
* stable layouts
* restrained motion

Avoid:

* excessive spinners
* flashing content
* dramatic loading animations
* unnecessary progress bars
* layout jumps

The interface should feel **fast and polished**, not overloaded with indicators.

---

# Request Lifecycle Standard

When implementing an asynchronous operation, think through the complete lifecycle:

```text
idle
↓
request begins
↓
loading
↓
success
```

or:

```text
idle
↓
request begins
↓
loading
↓
error
↓
retry
↓
success
```

For cached data:

```text
cached
↓
display cached
↓
background refresh
↓
updated
```

For optimistic mutation:

```text
current state
↓
optimistic state
↓
request
↓
confirmed
```

or:

```text
current state
↓
optimistic state
↓
request fails
↓
rollback
```

For streaming:

```text
idle
↓
connecting
↓
streaming
↓
partial content
↓
complete
```

or:

```text
streaming
↓
interrupted
↓
error/recovery
```

Thinking through the entire lifecycle prevents incomplete implementations.

---

# Implementation Checklist

Before adding a data-performance pattern:

* [ ] What actual problem does it solve?
* [ ] Is that problem present?
* [ ] Is the pattern necessary?
* [ ] Is there a simpler solution?
* [ ] Does it improve user experience?
* [ ] Does it reduce unnecessary work?
* [ ] Does it increase complexity?
* [ ] Does it introduce synchronization concerns?
* [ ] Does it introduce race conditions?
* [ ] Does it require cleanup?
* [ ] Does it affect security?
* [ ] Does it affect data freshness?
* [ ] Does it affect mobile bandwidth?
* [ ] Does it affect battery usage?
* [ ] Does it require special error handling?
* [ ] Does it require cache invalidation?
* [ ] Does it require cancellation?
* [ ] Does it require retry limits?
* [ ] Can it create duplicate mutations?

---

# Request Checklist

For API requests:

* [ ] Request identity is correct
* [ ] Relevant parameters are included
* [ ] Duplicate requests are avoided where appropriate
* [ ] Obsolete requests can be cancelled
* [ ] Race conditions are handled
* [ ] Initial loading is represented
* [ ] Refreshing is distinguished from initial loading
* [ ] Empty state is handled
* [ ] Error state is handled
* [ ] Retry behavior is appropriate
* [ ] Retry limits exist
* [ ] Timeout behavior is appropriate
* [ ] Sensitive data is handled safely
* [ ] Cleanup occurs when the request is no longer needed

---

# Cache Checklist

When implementing caching:

* [ ] Data actually benefits from caching
* [ ] Cache keys uniquely identify the data
* [ ] Relevant parameters are included
* [ ] Freshness requirements are understood
* [ ] Cache lifetime is appropriate
* [ ] Sensitive data is handled carefully
* [ ] Logout behavior is considered
* [ ] User switching is considered
* [ ] Mutation invalidation is defined
* [ ] Cache size is reasonable
* [ ] Stale data is safe to display
* [ ] Background refresh behavior is defined

---

# Optimistic Update Checklist

Before using optimistic updates:

* [ ] Operation is predictable
* [ ] Operation usually succeeds
* [ ] Immediate feedback is valuable
* [ ] Rollback is straightforward
* [ ] Previous state can be preserved
* [ ] Concurrent mutations are considered
* [ ] Server response is reconciled
* [ ] Failure is clearly communicated
* [ ] Critical operations are not being treated casually

---

# Retry Checklist

For retry behavior:

* [ ] Failure can actually be transient
* [ ] Permanent errors are excluded
* [ ] Maximum attempts exist
* [ ] Exponential backoff is used where appropriate
* [ ] Jitter is considered
* [ ] Rate limiting is respected
* [ ] User-visible operations provide appropriate feedback
* [ ] Mutation idempotency is considered
* [ ] Infinite retry loops are impossible

---

# Polling Checklist

For polling:

* [ ] Polling is actually necessary
* [ ] Refresh frequency is justified
* [ ] Polling stops when no longer needed
* [ ] Polling pauses or slows when appropriate
* [ ] Failed requests back off
* [ ] Duplicate polling requests are prevented
* [ ] Component cleanup exists
* [ ] Server load is acceptable
* [ ] Mobile battery/network impact is considered
* [ ] Streaming is not a better fit

---

# Streaming Checklist

For streaming:

* [ ] Partial results provide real value
* [ ] Streaming is actually needed
* [ ] Connection lifecycle is handled
* [ ] Loading state is clear
* [ ] Partial content is useful
* [ ] Cancellation is possible where appropriate
* [ ] Errors after partial content are handled
* [ ] Cleanup occurs
* [ ] Completion is explicit
* [ ] Streaming is not being used for a tiny response unnecessarily

---

# Pagination Checklist

For large datasets:

* [ ] Dataset size justifies pagination
* [ ] Backend supports appropriate pagination
* [ ] Loading state exists
* [ ] Empty state exists
* [ ] Error state exists
* [ ] Duplicate page loads are prevented
* [ ] Current page is preserved appropriately
* [ ] Cache behavior is understood
* [ ] Filters reset pagination appropriately
* [ ] Sorting is consistent with pagination

---

# Search Checklist

For server-side search:

* [ ] Input is debounced where appropriate
* [ ] Obsolete requests are cancelled
* [ ] Race conditions are prevented
* [ ] Empty search state is handled
* [ ] Loading state is clear
* [ ] Empty results are distinct from errors
* [ ] Search caching is limited appropriately
* [ ] Results are stable while the user interacts

---

# Mobile Checklist

For mobile applications:

* [ ] Network usage is controlled
* [ ] Polling is conservative
* [ ] Background activity is limited
* [ ] Battery usage is considered
* [ ] Payload size is reasonable
* [ ] Large datasets are paginated
* [ ] Images are optimized
* [ ] Offline behavior is intentional
* [ ] Failed requests recover gracefully
* [ ] User data is not unnecessarily persisted

---

# Final Decision Framework

Before implementing an advanced data pattern, ask:

## Question 1

**What problem am I solving?**

If there is no clear answer:

```text
Do not implement it.
```

---

## Question 2

**Can a simpler implementation solve it?**

If yes:

```text
Use the simpler implementation.
```

---

## Question 3

**What does the user gain?**

Possible answers:

* faster navigation
* fewer loading interruptions
* better responsiveness
* more reliable requests
* fresher data
* reduced bandwidth
* useful progress
* smoother search

If there is no meaningful user or system benefit, reconsider the pattern.

---

## Question 4

**What new failure modes does this introduce?**

Consider:

* race conditions
* stale data
* rollback failure
* duplicate mutations
* cache inconsistency
* retry loops
* memory leaks
* background requests
* authentication problems

Do not implement a pattern without understanding its failure behavior.

---

## Question 5

**What happens when the network fails?**

Every asynchronous feature should have an answer.

Do not design only for:

```text
request succeeds
```

Design for:

```text
slow
failed
cancelled
stale
offline
partial
```

when relevant.

---

# Recommended Defaults

When no special requirement exists:

## Basic API request

Use:

```text
loading
+
success
+
empty
+
error
```

---

## CRUD

Consider:

```text
server state
+
cache
+
request deduplication
+
targeted invalidation
```

Add optimistic updates only when appropriate.

---

## Search

Use:

```text
debounce
+
cancellation
+
race-condition protection
```

Add short-lived caching if repeated searches benefit from it.

---

## Dashboard

Use:

```text
cache
+
background refresh
```

Add polling only when freshness requirements justify it.

---

## Live dashboard

Use:

```text
smart polling
```

or:

```text
SSE
```

or:

```text
WebSocket
```

depending on the actual requirement.

---

## Long-running process

Use:

```text
job state
+
progress or polling
```

or streaming when partial output is useful.

---

## Large data

Use:

```text
pagination
```

and consider:

```text
prefetch
+
virtualization
```

only when useful.

---

# Final Principles

When implementing frontend data behavior:

**Use the simplest strategy that satisfies the requirement.**

**Do not implement every performance technique by default.**

**Optimize user-perceived performance, not only network metrics.**

**Avoid unnecessary requests.**

**Deduplicate requests when duplicate requests actually occur.**

**Cancel requests that are no longer relevant.**

**Debounce high-frequency input.**

**Throttle continuous browser events when appropriate.**

**Cache data when repeated access provides meaningful benefit.**

**Use stale-while-revalidate when temporary staleness is safe and faster feedback is valuable.**

**Invalidate only the data affected by a mutation whenever practical.**

**Use optimistic updates only when immediate feedback is valuable and rollback is safe.**

**Treat the server as authoritative.**

**Retry transient failures, not permanent failures.**

**Use exponential backoff for repeated retries.**

**Use jitter when synchronized retries could create traffic spikes.**

**Never retry forever.**

**Never blindly retry non-idempotent mutations.**

**Poll only when periodic freshness is actually required.**

**Make polling adaptive when application state allows it.**

**Stop polling when the data is no longer needed.**

**Use streaming when partial results provide meaningful value.**

**Do not stream small responses simply because streaming is available.**

**Use SSE when one-way server-to-client updates are sufficient.**

**Use WebSockets only when genuine bidirectional real-time communication is required.**

**Use pagination for large datasets.**

**Use infinite scrolling only when it fits the user's workflow.**

**Use virtualization only when rendering performance justifies it.**

**Prefetch when future user intent is reasonably predictable.**

**Do not aggressively prefetch data the user probably will never use.**

**Preserve existing useful content during background refresh.**

**Distinguish initial loading from refreshing.**

**Distinguish empty states from error states.**

**Give mutations clear feedback.**

**Prevent accidental duplicate submissions.**

**Protect active user input from background synchronization.**

**Do not let cached state become a substitute for authorization.**

**Do not allow private cached data to leak across users.**

**Clean up requests, timers, streams, subscriptions, and listeners.**

**Consider mobile bandwidth and battery usage.**

**Respect accessibility and reduced-motion preferences.**

**Measure meaningful problems before introducing significant optimization.**

**Do not optimize imaginary problems.**

The final frontend data layer should feel:

**Fast. Responsive. Reliable. Predictable. Efficient.**

The user should experience a polished interface where data appears at the right time, updates when necessary, recovers gracefully from failures, and never feels unnecessarily complicated.

The implementation may be sophisticated internally when the project requires it, but the user experience should remain simple.

```
```
