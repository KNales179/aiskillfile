````md
---
name: backend-api-design
description: Design clear, consistent, maintainable, secure, predictable, and scalable backend APIs. Use this skill whenever creating, redesigning, reviewing, or extending API endpoints, routes, controllers, services, request and response formats, validation, error handling, pagination, filtering, sorting, searching, API versioning, resource relationships, status codes, idempotency, documentation, or backend architecture. Prefer practical API designs appropriate to the project's actual size and requirements rather than unnecessary enterprise complexity.
---

# Backend API Design

Design APIs that are predictable for clients, easy to maintain for developers, safe to evolve, and appropriate for the project's actual requirements.

A good API should make it obvious:

- what resource is being accessed
- what operation is being performed
- what data the client should send
- what data the server will return
- what errors can occur
- what authentication is required
- what authorization is required
- how the client should handle pagination
- how filtering and sorting work
- whether an operation can safely be retried
- whether an operation is synchronous or asynchronous

The API should feel like a coherent system rather than a collection of endpoints that were added one at a time.

Prioritize:

1. Consistency
2. Predictability
3. Correct HTTP semantics
4. Clear resource boundaries
5. Validation
6. Useful error responses
7. Security
8. Maintainability
9. Evolvability
10. Appropriate performance

Do not design an API to be complicated simply because large systems use complicated API architectures.

A small application can have an excellent API without introducing unnecessary layers, protocols, or abstractions.

---

# Core Principles

## Design around resources and capabilities

Start by identifying the actual resources and operations in the system.

For example:

```text
Drivers
Vehicles
Franchises
Enforcers
Violations
Transactions
Admins
Uploads
````

Then determine what the client needs to do with them.

Typical operations include:

```text
Create
Read
Update
Delete
Search
Filter
Sort
Assign
Approve
Reject
Archive
Restore
Upload
Export
```

Do not create endpoints simply because a database model exists.

An API should represent meaningful application behavior.

---

# Consistent URL Structure

Use predictable route structures.

For example:

```text
GET    /api/v1/drivers
GET    /api/v1/drivers/:id
POST   /api/v1/drivers
PATCH  /api/v1/drivers/:id
DELETE /api/v1/drivers/:id
```

The exact structure can vary by project, but consistency is more important than blindly following a particular convention.

Avoid unrelated naming styles such as:

```text
/api/getDrivers
/api/driver-list
/api/createNewDriver
/api/update-driver-data
```

when the rest of the API uses resource-oriented routes.

Prefer one coherent convention.

---

# HTTP Methods

Use HTTP methods according to their intended meaning.

## GET

Use `GET` for retrieving data.

Examples:

```text
GET /api/v1/drivers
GET /api/v1/drivers/123
GET /api/v1/violations
```

Do not use `GET` for operations that modify server state.

Avoid:

```text
GET /api/v1/deleteDriver/123
```

---

## POST

Use `POST` when creating a resource or performing an operation that does not fit a simple resource update.

Examples:

```text
POST /api/v1/drivers
POST /api/v1/auth/login
POST /api/v1/transactions
```

POST is also appropriate for certain actions such as:

```text
POST /api/v1/reports/export
```

when the operation creates a job or performs a server-side action.

---

## PUT

Use `PUT` when replacing a resource representation where that semantic is appropriate.

For example:

```text
PUT /api/v1/drivers/123
```

If the project does not need full replacement semantics, do not use `PUT` simply because it exists.

---

## PATCH

Use `PATCH` for partial updates.

Example:

```text
PATCH /api/v1/drivers/123
```

with:

```json
{
  "status": "INACTIVE"
}
```

This is generally preferable to requiring the client to submit the entire driver object for a small change.

---

## DELETE

Use `DELETE` when removing a resource.

Example:

```text
DELETE /api/v1/drivers/123
```

However, do not physically delete records when the business requirement calls for archival or deactivation.

For example, a driver may need:

```text
status = INACTIVE
```

instead of being permanently deleted.

---

# Avoid RPC-Style APIs by Default

Avoid creating endpoints that look like frontend functions:

```text
/api/createDriver
/api/updateDriver
/api/deleteDriver
/api/getDriver
```

Prefer resource-oriented structures:

```text
POST   /drivers
PATCH  /drivers/:id
DELETE /drivers/:id
GET    /drivers/:id
```

However, actions that represent genuine business operations can use action-oriented endpoints when appropriate.

For example:

```text
POST /transactions/:id/approve
POST /transactions/:id/reject
POST /rides/:id/cancel
```

These are meaningful domain actions rather than generic CRUD wrappers.

---

# Naming Conventions

Choose one naming convention and use it consistently.

For URLs, lowercase plural nouns are generally a strong default:

```text
/drivers
/enforcers
/violations
/transactions
/vehicles
/franchises
```

Avoid mixing:

```text
/drivers
/Enforcer
/vehicle-record
/transactionList
```

Consistency makes APIs easier to understand and maintain.

---

# API Versioning

Version APIs when there is a meaningful compatibility boundary.

A common approach is:

```text
/api/v1/drivers
```

Then a future breaking version may become:

```text
/api/v2/drivers
```

Do not create a new version for every minor change.

Version when the existing contract can no longer safely serve existing clients.

---

# Breaking vs Non-Breaking Changes

Treat API compatibility as an explicit concern.

Potentially breaking changes include:

* removing fields
* renaming fields
* changing field types
* changing required request fields
* changing response structure
* changing authentication requirements
* changing semantics of existing fields
* removing endpoints
* changing pagination behavior unexpectedly

Potentially non-breaking changes include:

* adding optional request fields
* adding new endpoints
* adding new response fields when clients tolerate unknown fields
* adding new enum values only when clients are designed to handle them

Consider how existing clients behave before changing the API contract.

---

# Request Validation

Validate all untrusted input at the backend.

Never assume that frontend validation is sufficient.

Validate:

* required fields
* data types
* string lengths
* numeric ranges
* enum values
* identifiers
* dates
* nested objects
* arrays
* file metadata
* query parameters
* pagination parameters
* filtering parameters

Example:

```json
{
  "full_name": "John Doe",
  "status": "REGISTERED"
}
```

The backend should verify that:

```text
full_name
→ valid string

status
→ allowed enum value
```

Do not simply pass request bodies directly into database operations.

---

# Validate at the Boundary

A useful architecture is:

```text
Request
   ↓
Authentication
   ↓
Authorization
   ↓
Validation
   ↓
Controller
   ↓
Service / Business Logic
   ↓
Database
```

Validation should happen before invalid data reaches business logic or persistence.

---

# Do Not Trust Client-Supplied Sensitive Fields

Do not allow clients to freely control fields that should be determined by the server.

Examples:

```text
createdBy
createdAt
updatedAt
role
permissions
approval status
verified status
account ownership
```

Instead of trusting:

```json
{
  "role": "SUPER_ADMIN"
}
```

derive sensitive values from authenticated identity and authorized server-side logic.

---

# Authentication vs Authorization

Do not treat authentication as authorization.

Authentication answers:

```text
Who are you?
```

Authorization answers:

```text
Are you allowed to do this?
```

For example:

```text
GET /drivers/123
```

may require authentication.

But:

```text
DELETE /drivers/123
```

may additionally require:

```text
SUPER_ADMIN
```

or another appropriate permission.

Every protected endpoint should have an intentional authorization policy.

---

# Consistent Response Structure

Choose a response structure and use it consistently.

For example:

```json
{
  "success": true,
  "message": "Driver retrieved successfully",
  "data": {
    "id": "123",
    "name": "John Doe"
  }
}
```

For lists:

```json
{
  "success": true,
  "data": [
    {
      "id": "123",
      "name": "John Doe"
    }
  ]
}
```

The exact structure can differ.

The important requirement is consistency.

Do not have one endpoint return:

```json
{
  "data": {}
}
```

another return:

```json
{
  "result": {}
}
```

and another return:

```json
{
  "driver": {}
}
```

without a deliberate reason.

---

# Response Messages

Messages should be useful and concise.

Prefer:

```text
Driver updated successfully.
```

over:

```text
Operation completed.
```

For errors:

```text
Driver not found.
```

is more useful than:

```text
Something went wrong.
```

Do not expose sensitive implementation details.

Avoid returning:

```text
MongoServerError:
E11000 duplicate key error collection...
```

to clients.

Log technical details internally and return an appropriate API error.

---

# HTTP Status Codes

Use status codes intentionally.

Common examples:

```text
200 OK
201 Created
202 Accepted
204 No Content

400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Content
429 Too Many Requests

500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

Do not return `200 OK` for every situation.

---

# 200 OK

Use when the request succeeded and the response contains the result.

Example:

```text
GET /drivers/123
```

returns:

```text
200 OK
```

---

# 201 Created

Use when a resource was successfully created.

Example:

```text
POST /drivers
```

returns:

```text
201 Created
```

when a new driver was created.

---

# 202 Accepted

Use when the request has been accepted for processing but has not completed yet.

This is particularly useful for:

```text
large exports
report generation
document processing
background jobs
long-running operations
```

Example:

```text
POST /reports/export
```

returns:

```json
{
  "success": true,
  "message": "Export queued",
  "data": {
    "jobId": "abc123"
  }
}
```

with:

```text
202 Accepted
```

---

# 204 No Content

Use when the operation succeeds but there is intentionally no response body.

For example:

```text
DELETE /drivers/123
```

may return:

```text
204 No Content
```

when appropriate.

---

# 400 Bad Request

Use for malformed or invalid requests when appropriate.

Examples:

* malformed JSON
* invalid request structure
* invalid parameter format

---

# 401 Unauthorized

Use when authentication is missing or invalid.

Examples:

```text
missing token
expired token
invalid token
```

Do not use `401` simply because the user lacks permission.

---

# 403 Forbidden

Use when the server understands the identity but the user is not allowed to perform the operation.

Example:

```text
authenticated STAFF
        ↓
DELETE /drivers/123
        ↓
403 Forbidden
```

---

# 404 Not Found

Use when the requested resource does not exist or should not be exposed as existing according to the application's security model.

Example:

```text
GET /drivers/does-not-exist
```

---

# 409 Conflict

Use when the request conflicts with the current state of the resource.

Examples:

```text
duplicate username
duplicate transaction
invalid state transition
optimistic concurrency conflict
resource already processed
```

---

# 422 Unprocessable Content

Use when the request structure is valid but the submitted data cannot be processed according to application rules.

For example:

```text
fare amount is invalid
date range is impossible
status transition is not allowed
```

Use consistently with the project's chosen validation strategy.

---

# 429 Too Many Requests

Use when rate limits are exceeded.

The response may include information that helps the client determine when it can try again.

Do not rely solely on frontend throttling for API abuse prevention.

---

# 500 Internal Server Error

Use for unexpected server-side failures.

Do not expose internal stack traces to clients.

Log the detailed error internally.

---

# Error Response Design

Use a consistent error structure.

For example:

```json
{
  "success": false,
  "message": "Validation failed",
  "error": {
    "code": "VALIDATION_ERROR",
    "details": [
      {
        "field": "email",
        "message": "A valid email address is required"
      }
    ]
  }
}
```

A simpler project may use:

```json
{
  "success": false,
  "message": "Driver not found"
}
```

The level of detail should match the project's requirements.

Do not create complicated error schemas without a real need.

---

# Machine-Readable Error Codes

For applications with substantial frontend logic, use stable error codes.

Examples:

```text
VALIDATION_ERROR
AUTHENTICATION_REQUIRED
FORBIDDEN
RESOURCE_NOT_FOUND
DUPLICATE_RESOURCE
CONFLICT
RATE_LIMITED
INTERNAL_ERROR
```

The frontend can then respond based on:

```text
error.code
```

rather than parsing human-readable messages.

Messages can change.

Stable error codes provide a stronger API contract.

---

# Validation Error Details

When multiple fields are invalid, provide structured details when useful.

Example:

```json
{
  "success": false,
  "message": "Validation failed",
  "error": {
    "code": "VALIDATION_ERROR",
    "fields": {
      "username": "Username is required",
      "email": "Invalid email address",
      "password": "Password must contain at least 8 characters"
    }
  }
}
```

This allows the frontend to display field-level errors correctly.

---

# Pagination

Never return an unbounded amount of database data by default.

Avoid:

```text
GET /drivers
```

returning tens of thousands of records.

Use pagination for potentially large collections.

For example:

```text
GET /api/v1/drivers?page=1&limit=20
```

or another appropriate pagination strategy.

---

# Limit Pagination Size

Do not trust arbitrary client-provided limits.

Avoid allowing:

```text
?limit=999999999
```

Set a maximum.

For example:

```text
default limit = 20
maximum limit = 100
```

The actual values should depend on the project.

---

# Offset Pagination

Offset pagination can be simple and appropriate for administrative systems.

Example:

```text
?page=2&limit=20
```

Response:

```json
{
  "success": true,
  "data": [],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 250,
    "totalPages": 13
  }
}
```

Use it when the dataset and navigation requirements make it appropriate.

---

# Cursor Pagination

Cursor pagination can be more appropriate for:

* very large datasets
* feeds
* frequently changing data
* infinite scrolling
* high-volume systems

Example:

```text
GET /drivers?limit=20&cursor=abc123
```

Response:

```json
{
  "success": true,
  "data": [],
  "pagination": {
    "nextCursor": "xyz789",
    "hasMore": true
  }
}
```

Do not implement cursor pagination simply because it sounds more advanced.

---

# Sorting

Support sorting intentionally.

Example:

```text
GET /drivers?sortBy=createdAt&sortOrder=desc
```

Never blindly pass arbitrary client values directly into database query construction.

Use an allowlist.

For example:

```js
const allowedSortFields = [
  "createdAt",
  "fullName",
  "status"
];
```

---

# Filtering

Filtering should be predictable.

Example:

```text
GET /drivers?status=REGISTERED
```

Multiple filters may be supported:

```text
GET /drivers?status=REGISTERED&franchiseType=REGULAR
```

Validate filter values.

Do not expose arbitrary database operators through query parameters.

Avoid allowing clients to construct unrestricted database queries.

---

# Searching

Search parameters should have clear semantics.

Example:

```text
GET /drivers?search=Juan
```

The backend should define what fields are searched.

For example:

```text
name
license number
plate number
driver code
```

Do not silently search every database field.

For large datasets, consider appropriate indexes or search infrastructure.

---

# Query Parameter Validation

Validate:

```text
page
limit
cursor
sortBy
sortOrder
search
filters
date ranges
```

Examples:

```text
page >= 1
limit >= 1
limit <= maximum
sortOrder ∈ {asc, desc}
```

Reject invalid values rather than allowing unpredictable database behavior.

---

# Date Filtering

Use a consistent date format.

Prefer standard representations such as ISO 8601.

Example:

```text
?from=2026-08-01T00:00:00Z
&to=2026-08-31T23:59:59Z
```

Be explicit about timezone behavior.

Do not assume that all clients use the same local timezone.

---

# Resource Relationships

Represent relationships clearly.

For example:

```text
Driver
   ↓
Vehicle
   ↓
Franchise
```

Do not return enormous nested structures by default.

Avoid:

```text
Driver
 └── Vehicle
      └── Franchise
           └── Drivers
                └── Vehicles
                     └── ...
```

This can produce:

* excessive payloads
* circular structures
* unnecessary database queries
* difficult frontend state management

Return only what the client needs.

---

# Nested Routes

Nested routes can represent strong ownership relationships.

For example:

```text
GET /drivers/:driverId/vehicles
```

can make sense if vehicles are meaningfully scoped to a driver.

But avoid excessive nesting:

```text
/drivers/:driverId/vehicles/:vehicleId/franchises/:franchiseId/transactions/:transactionId
```

Deeply nested URLs often become difficult to understand.

Keep relationships explicit but manageable.

---

# Response Expansion

If clients sometimes need related resources, consider controlled expansion.

For example:

```text
GET /drivers/123?include=vehicle
```

or another project-appropriate mechanism.

Do not automatically return every related resource on every request.

---

# Field Selection

For large resources, field selection can sometimes reduce payload size.

For example:

```text
GET /drivers?fields=id,fullName,status
```

Only implement this when it provides meaningful value.

Do not add field-selection syntax merely because sophisticated APIs support it.

---

# DTOs and Response Mapping

Do not automatically return raw database documents.

Create an API representation when necessary.

Database model:

```text
_internalId
passwordHash
createdBy
internalFlags
```

API response:

```json
{
  "id": "123",
  "fullName": "John Doe",
  "status": "REGISTERED"
}
```

This protects internal fields and gives the API control over its public contract.

---

# Never Return Sensitive Fields

Never accidentally expose fields such as:

```text
passwordHash
refreshToken
resetToken
verificationToken
internal secrets
private keys
security answers
sensitive internal metadata
```

The response layer should explicitly control what leaves the backend.

---

# Controllers and Services

For projects that benefit from separation, a useful structure is:

```text
Route
   ↓
Controller
   ↓
Service
   ↓
Model / Repository
```

Routes should primarily define:

* URL
* HTTP method
* middleware
* controller handler

Controllers should primarily handle:

* request extraction
* validation coordination
* calling business logic
* response formatting

Services should handle:

* business rules
* workflows
* transactions
* domain operations

Do not blindly create a service layer for trivial CRUD if it provides no value.

---

# Avoid Giant Controllers

Avoid controllers containing:

```text
validation
authentication
business rules
database queries
file uploads
email sending
external APIs
response formatting
```

all in one enormous function.

Break meaningful responsibilities apart.

However, do not split every three lines into a new file.

The goal is maintainability, not file-count optimization.

---

# Routes and Middleware

Routes should clearly show middleware requirements.

For example:

```js
router.get(
  "/",
  authenticate,
  authorize("DRIVER_VIEW"),
  getDrivers
);
```

This makes the endpoint's protection easy to understand.

Avoid hiding critical authorization logic deep inside unrelated code where possible.

---

# Middleware Scope

Middleware should apply only where intended.

Prefer:

```js
app.use("/api/drivers", driverRoutes);
app.use("/api/enforcers", enforcerRoutes);
```

with route-specific middleware inside the appropriate router.

For example:

```js
driverRouter.delete(
  "/:id",
  authenticate,
  authorize("DRIVER_DELETE"),
  deleteDriver
);
```

Do not accidentally attach security middleware globally when only one route requires it.

---

# Router-Level Middleware

Router-level middleware is useful when every endpoint under a router shares the same requirement.

Example:

```js
router.use(authenticate);
```

Then:

```text
/drivers
/drivers/:id
/drivers/:id/vehicles
```

all require authentication.

Use router-level middleware when the requirement genuinely applies to the entire resource.

Use route-level middleware when only specific endpoints require additional protection.

---

# Business Actions

Some operations are not ordinary CRUD.

Examples:

```text
POST /transactions/:id/approve
POST /transactions/:id/reject
POST /drivers/:id/impound
POST /drivers/:id/restore
POST /users/:id/verify
```

These endpoints can represent explicit business actions.

Do not force every business operation into a generic:

```text
PATCH /resource/:id
```

if doing so makes the API ambiguous.

---

# State Transition Endpoints

For important state transitions, explicit action endpoints can improve clarity.

For example:

```text
POST /violations/:id/void
```

can be clearer than:

```text
PATCH /violations/:id

{
  "status": "VOID"
}
```

when `void` represents a significant business operation with special authorization or auditing requirements.

---

# Authorization for Actions

Business actions should have explicit authorization.

For example:

```text
POST /transactions/:id/approve
```

may require:

```text
ADMIN
```

while:

```text
POST /transactions/:id/reject
```

may require another permission.

Do not assume that because the resource itself is accessible, every action on it is allowed.

---

# Idempotency in API Design

Design APIs with retries in mind.

Potentially dangerous operations include:

```text
payment
booking
transaction
resource creation
assignment
notification
external service action
```

Consider idempotency keys when repeated requests could create duplicate effects.

Example:

```http
POST /api/v1/transactions
Idempotency-Key: 4d8f...
```

The concurrency and idempotency implementation belongs to the backend, not the frontend alone.

---

# Long-Running Operations

Do not keep an HTTP request open unnecessarily for long operations.

Instead:

```text
POST /reports/export
        ↓
202 Accepted
        ↓
job created
        ↓
worker processes
        ↓
GET /reports/jobs/:jobId
```

or another appropriate job-status mechanism.

Use this when processing genuinely takes long enough to justify asynchronous execution.

Do not introduce queues for operations that consistently finish quickly.

---

# File Upload APIs

File uploads should have intentional endpoints.

For example:

```text
POST /api/v1/uploads
```

or a resource-specific endpoint:

```text
POST /api/v1/violations/:id/receipt
```

Validate:

* file type
* file size
* filename handling
* ownership
* authorization
* storage destination
* upload status

Do not trust client-provided MIME types or filenames blindly.

---

# API and Cloud Storage

When using external storage such as image/document storage:

```text
Client
  ↓
API
  ↓
Storage
```

define what happens if:

```text
upload succeeds
database update fails
```

or:

```text
database update succeeds
storage upload fails
```

Use appropriate cleanup, retry, or asynchronous processing strategies.

Do not assume two independent systems share a transaction.

---

# Export APIs

Large exports should not necessarily be synchronous.

Avoid:

```text
GET /drivers/export
```

generating a massive file during the request if it could take a long time.

Consider:

```text
POST /drivers/export
```

returning:

```json
{
  "success": true,
  "data": {
    "jobId": "abc123"
  }
}
```

Then:

```text
GET /exports/abc123
```

can expose job status.

Use synchronous exports when the data is small and processing is reliably fast.

---

# API Security

API design and security are connected.

Every API should consider:

* authentication
* authorization
* input validation
* rate limiting
* payload limits
* request size limits
* sensitive data exposure
* injection prevention
* CORS configuration
* secure headers
* token handling
* audit logging where necessary

Security should not be treated as an afterthought added after the routes already exist.

---

# Mass Assignment

Avoid blindly passing request bodies into database updates.

Dangerous pattern:

```js
Model.findByIdAndUpdate(id, req.body);
```

A client may attempt to modify fields that should never be client-controlled.

Prefer explicit field selection.

For example:

```js
const { fullName, email, phone } = req.body;

await Model.findByIdAndUpdate(id, {
  fullName,
  email,
  phone
});
```

The exact implementation may differ, but the principle is:

**only allow fields that the endpoint is intended to modify.**

---

# API Rate Limits

Different endpoints may require different limits.

For example:

```text
Login
→ strict

Password reset
→ strict

Public search
→ moderate

Normal authenticated CRUD
→ normal

Internal health check
→ appropriate internal policy
```

Do not necessarily apply one identical rate limit to every endpoint.

---

# Request Size Limits

Limit payload sizes.

Consider limits for:

* JSON bodies
* file uploads
* query strings
* multipart forms

This protects server resources and reduces abuse opportunities.

---

# CORS

Configure CORS intentionally.

Do not automatically use:

```js
origin: "*"
```

for private administrative APIs.

Allow only the origins required by the project when credentials or authenticated browser requests are involved.

---

# API Documentation

Document endpoints sufficiently for frontend developers and future maintainers.

At minimum, document:

```text
Method
Path
Authentication
Authorization
Parameters
Request body
Response
Errors
Pagination
Special behavior
```

For example:

```text
POST /api/v1/drivers

Authentication:
Required

Authorization:
DRIVER_CREATE

Body:
{
  "fullName": "John Doe",
  "licenseNo": "..."
}

Responses:
201 Created
400 Validation Error
401 Authentication Required
403 Forbidden
409 Conflict
```

---

# OpenAPI

For larger APIs, consider maintaining an OpenAPI specification.

It can document:

* routes
* schemas
* parameters
* authentication
* responses
* error structures

Do not introduce OpenAPI tooling solely for a tiny prototype unless it provides meaningful value.

For a larger production system, keeping the API contract documented can significantly improve maintainability.

---

# Contract Consistency

The API should behave consistently across resources.

For example:

```text
GET /drivers
GET /vehicles
GET /enforcers
```

should ideally share conventions for:

* pagination
* filtering
* sorting
* response format
* errors

Do not make every resource invent its own query syntax.

---

# Empty Collections

A successful collection request with no results should generally still be successful.

For example:

```text
GET /drivers?status=REGISTERED
```

may return:

```json
{
  "success": true,
  "data": [],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 0,
    "totalPages": 0
  }
}
```

Do not treat "no matching records" as a server error.

---

# Single Resource vs Collection

Distinguish between:

```text
GET /drivers
```

and:

```text
GET /drivers/:id
```

A collection with no records is usually:

```text
200 OK
```

A specific resource that does not exist is usually:

```text
404 Not Found
```

This distinction makes client behavior predictable.

---

# Soft Delete and Archiving

Do not automatically expose permanent deletion.

If records have historical or legal significance, consider:

```text
status
archivedAt
deletedAt
```

or another appropriate lifecycle mechanism.

For example:

```text
REGISTERED
INACTIVE
IMPOUNDED
ARCHIVED
```

may represent meaningful business states.

Do not destroy historical data merely because a frontend button says "Delete."

---

# Auditability

Important administrative actions should be traceable when required.

Consider recording:

```text
who
what
when
resource
previous state
new state
reason
```

Examples:

```text
Admin approved transaction
Admin changed driver status
Admin uploaded receipt
Admin deleted record
Admin changed permissions
```

Do not add audit logging to every insignificant read unless required.

---

# API Evolution

When changing an API:

1. Identify affected clients.
2. Determine whether the change is breaking.
3. Prefer backward-compatible changes when practical.
4. Introduce a version when necessary.
5. Document the change.
6. Provide migration guidance when required.
7. Remove deprecated behavior deliberately.

Do not silently break existing clients.

---

# Deprecation

When an endpoint must eventually disappear:

```text
old endpoint
    ↓
deprecated
    ↓
migration period
    ↓
new endpoint
    ↓
old endpoint removed
```

Avoid abruptly removing endpoints that existing clients still depend on unless the project is small enough that the clients can be updated together.

---

# API Consistency Over Personal Preference

Do not redesign an API simply because another naming convention looks cleaner.

If the project already has:

```text
/api/v1/drivers
```

and:

```text
/api/v1/enforcers
```

continue the established convention unless there is a strong reason to change it.

Consistency across the project is often more valuable than theoretical perfection.

---

# API Design for Mobile Clients

Mobile clients may operate under:

* slow networks
* unstable connections
* limited bandwidth
* intermittent connectivity
* repeated requests
* outdated application versions

Therefore:

* keep payloads reasonable
* paginate large collections
* provide useful errors
* support retry-safe operations where appropriate
* avoid unnecessarily chatty APIs
* consider caching
* avoid requiring many sequential requests for basic screens

Do not assume the client has a perfect connection.

---

# API Design for Administrative Systems

Administrative systems often require:

* filtering
* sorting
* searching
* pagination
* exports
* detailed records
* role-based authorization
* audit trails
* status transitions
* document uploads

Design these capabilities intentionally.

For example:

```text
GET /drivers
```

may support:

```text
?page=1
&limit=20
&search=Juan
&status=REGISTERED
&sortBy=createdAt
&sortOrder=desc
```

Do not create separate endpoints for every possible filter combination.

---

# Avoid Endpoint Explosion

Do not create:

```text
/drivers/registered
/drivers/inactive
/drivers/colorum
/drivers/impounded
```

if the same result can be represented naturally through filtering:

```text
/drivers?status=REGISTERED
/drivers?status=INACTIVE
/drivers?status=COLORUM
/drivers?status=IMPOUNDED
```

However, create separate endpoints when the operation genuinely represents a different domain resource or workflow.

---

# Avoid Generic Mega-Endpoints

Do not create one endpoint that does everything.

Avoid:

```text
POST /api/manage
```

with:

```json
{
  "action": "updateDriver",
  "resource": "driver",
  ...
}
```

This makes:

* authorization harder
* validation harder
* documentation harder
* testing harder
* maintenance harder

Prefer explicit endpoints.

---

# API Layering

A practical backend may look like:

```text
Client
  ↓
HTTP Route
  ↓
Middleware
  ↓
Controller
  ↓
Service
  ↓
Database / External Services
```

Each layer should have a meaningful responsibility.

Do not introduce layers simply to increase the number of files.

---

# Controller Responsibility

Controllers should generally:

1. Receive the request.
2. Extract relevant input.
3. Invoke validation where appropriate.
4. Call business logic.
5. Translate the result into an HTTP response.
6. Handle expected API-level errors.

Avoid putting large business workflows directly into controllers.

---

# Service Responsibility

Services should contain meaningful business logic.

Examples:

```text
approveTransaction()
assignDriver()
registerVehicle()
completeViolation()
generateExport()
```

A service can coordinate:

```text
database
+
transactions
+
external APIs
+
business rules
```

when necessary.

---

# Database Layer

Keep database-specific operations organized.

Depending on project complexity, this may use:

```text
Model
Repository
Data access layer
Service
```

Do not create a repository layer merely because a tutorial says every application needs one.

Use it when it improves maintainability or abstraction.

---

# Avoid Over-Abstraction

Do not turn:

```text
simple CRUD
```

into:

```text
route
→ middleware
→ controller
→ service
→ use case
→ repository
→ adapter
→ provider
→ database
```

if the project does not benefit from it.

Architecture should follow complexity.

---

# API Contract First

Before implementing an endpoint, define:

```text
Method
Path
Authentication
Authorization
Request
Response
Errors
Side effects
```

For example:

```text
PATCH /api/v1/drivers/:id

Authentication:
Required

Authorization:
DRIVER_UPDATE

Request:
{
  "fullName": "John Doe"
}

Success:
200 OK

Errors:
400 Validation Error
401 Authentication Required
403 Forbidden
404 Not Found
409 Conflict
```

Then implement the endpoint.

This prevents the frontend and backend from inventing different assumptions.

---

# Frontend and Backend Agreement

The frontend should not have to guess:

```text
what status code means
what error structure looks like
what fields are required
what fields are optional
whether a request is asynchronous
whether pagination exists
```

Make the contract explicit.

---

# API Testing

Test more than successful requests.

For each important endpoint, test:

```text
valid request
invalid request
missing authentication
insufficient authorization
missing resource
duplicate request
duplicate resource
concurrent update
malformed input
large input
unexpected database failure
external dependency failure
```

For state-changing endpoints, test repeated execution.

---

# Contract Testing

When frontend and backend are developed separately, consider testing that the actual implementation matches the expected API contract.

Important areas include:

* field names
* field types
* required fields
* status codes
* error structures
* pagination
* authentication behavior

This reduces integration surprises.

---

# Logging

API logs should make important requests traceable.

Useful information may include:

```text
request ID
method
route
status
duration
authenticated user
resource ID
error code
```

Do not log sensitive information such as:

```text
passwords
tokens
secrets
full authentication credentials
```

---

# Request IDs

Assign a request identifier where useful.

Example:

```text
X-Request-ID: 8f31...
```

This can help connect:

```text
frontend request
        ↓
API log
        ↓
service log
        ↓
database/error log
```

Request IDs become particularly valuable as the backend grows.

---

# Performance-Aware API Design

API design affects performance.

Avoid:

```text
frontend
 ↓
GET /driver
 ↓
GET /vehicle
 ↓
GET /franchise
 ↓
GET /violations
 ↓
GET /transactions
```

when a screen genuinely needs all of that information and a better API design can provide it efficiently.

However, do not create enormous responses containing every possible relationship.

Find the appropriate balance.

---

# N+1 Requests

Be aware of both:

```text
N+1 frontend HTTP requests
```

and:

```text
N+1 database queries
```

For example:

```text
GET /drivers
```

followed by:

```text
GET /drivers/1/vehicle
GET /drivers/2/vehicle
GET /drivers/3/vehicle
...
```

may be inefficient.

Consider whether the API should return the required summary information or support an appropriate inclusion mechanism.

---

# Caching

Cache data when it is safe and beneficial.

Good candidates may include:

* relatively static metadata
* configuration
* public reference data
* expensive repeated queries
* frequently requested read-only resources

Do not cache sensitive or rapidly changing data without understanding consistency requirements.

---

# Cache Invalidation

When caching mutable resources, define how the cache becomes invalidated or refreshed.

Do not add caching without knowing:

```text
when is cached data stale?
```

A fast incorrect response is still incorrect.

---

# Public vs Private APIs

Identify whether an endpoint is:

```text
public
authenticated
internal
administrative
```

Different categories may require different:

* authentication
* rate limits
* response detail
* caching rules
* logging
* security controls

Do not expose administrative functionality through a public endpoint simply because the frontend hides the button.

---

# Internal Endpoints

Internal APIs are not automatically trusted.

If an endpoint can modify sensitive data, still enforce appropriate:

* authentication
* authorization
* validation
* network restrictions
* service identity
* auditing

"Only our frontend calls this endpoint" is not a security boundary.

---

# Webhooks

When accepting webhooks:

```text
POST /api/v1/webhooks/provider
```

consider:

* signature verification
* replay protection
* idempotency
* duplicate delivery
* timeout behavior
* asynchronous processing

Webhook providers may deliver the same event more than once.

Do not assume:

```text
one webhook = one delivery
```

---

# External API Integration

When your API calls another API:

```text
Client
 ↓
Your API
 ↓
External API
```

define:

* timeout
* retry behavior
* error mapping
* authentication
* rate limits
* response validation
* failure handling

Do not expose the external service's raw response directly unless that is deliberately part of your API contract.

---

# Error Mapping

Translate external errors into your own API's semantics where appropriate.

Do not automatically return:

```text
Cloudinary's exact error
```

or:

```text
third-party API response
```

to the frontend.

Instead:

```text
External service
      ↓
your backend
      ↓
your API error contract
```

This prevents external implementation details from becoming part of your public API.

---

# Backward Compatibility

When possible, new backend functionality should not require an immediate frontend rewrite.

Prefer:

```text
add optional field
```

over:

```text
rename existing field
```

when compatibility matters.

If a breaking change is necessary, communicate it explicitly.

---

# API Design Checklist

Before considering an endpoint complete:

* [ ] Does the URL clearly represent the resource or action?
* [ ] Is the HTTP method appropriate?
* [ ] Is the endpoint versioned appropriately?
* [ ] Is authentication required?
* [ ] Is authorization enforced?
* [ ] Is request input validated?
* [ ] Are sensitive fields protected?
* [ ] Is mass assignment prevented?
* [ ] Are status codes meaningful?
* [ ] Is the response structure consistent?
* [ ] Is the error structure consistent?
* [ ] Are machine-readable error codes useful?
* [ ] Is pagination required?
* [ ] Are filters validated?
* [ ] Are sorting fields allowlisted?
* [ ] Is search behavior defined?
* [ ] Are payload sizes controlled?
* [ ] Is the operation idempotent where necessary?
* [ ] Can duplicate requests cause problems?
* [ ] Can concurrent requests cause conflicts?
* [ ] Are state transitions validated?
* [ ] Are database constraints used where necessary?
* [ ] Are long-running operations asynchronous where appropriate?
* [ ] Are external dependencies handled safely?
* [ ] Are sensitive values excluded from logs?
* [ ] Is the endpoint documented?
* [ ] Is the endpoint tested for both success and failure?
* [ ] Does the endpoint follow existing project conventions?

---

# Anti-Patterns

Avoid:

* `/getUsers`
* `/createUser`
* `/updateUser`
* `/deleteUser`
* one giant `/manage` endpoint
* returning `200` for every response
* returning raw database documents
* exposing internal database errors
* trusting frontend validation
* trusting client-controlled roles
* allowing arbitrary database filters
* unlimited pagination
* arbitrary sorting fields
* unbounded response payloads
* returning sensitive fields
* deeply nested resources
* excessive endpoint duplication
* giant controllers
* unnecessary service/repository layers
* inconsistent response formats
* inconsistent error formats
* undocumented business actions
* silently breaking existing clients
* relying on frontend security
* assuming internal endpoints are automatically safe

---

# Design Decision Guide

Use this sequence when designing a new endpoint.

## Step 1 — Identify the resource

Ask:

```text
What does this endpoint operate on?
```

Example:

```text
drivers
vehicles
violations
transactions
```

---

## Step 2 — Identify the operation

Ask:

```text
Is this:
create
read
update
delete
search
filter
or a business action?
```

---

## Step 3 — Determine protection

Ask:

```text
Who can access it?
Who can modify it?
Who can perform the action?
```

---

## Step 4 — Define the request

Specify:

```text
path parameters
query parameters
headers
body
files
```

---

## Step 5 — Define validation

Determine:

```text
required fields
allowed values
limits
formats
relationships
business rules
```

---

## Step 6 — Define the response

Specify:

```text
success status
response body
pagination
metadata
```

---

## Step 7 — Define failure behavior

Specify:

```text
400
401
403
404
409
422
429
500
```

only where relevant.

Do not document every possible status code if the endpoint cannot realistically produce it.

---

## Step 8 — Consider concurrency

Ask:

```text
Can two requests modify this at once?
Can the request be repeated?
Can a worker process it twice?
```

If yes, coordinate with the `backend-concurrency` skill.

---

## Step 9 — Consider performance

Ask:

```text
Could this return a large amount of data?
Could this query become expensive?
Does this operation take a long time?
```

If yes, coordinate with the `backend-performance` or `backend-queues-workers` skill.

---

## Step 10 — Document the contract

Make the behavior clear enough that another developer can consume the endpoint without reading the implementation.

---

# Relationship With Other Backend Skills

This skill defines the API contract and structure.

It should work together with:

```text
backend-security
```

for:

```text
authentication
authorization
token handling
rate limiting
input security
```

Use:

```text
backend-concurrency
```

for:

```text
race conditions
transactions
idempotency
locking
duplicate requests
state transitions
```

Use:

```text
backend-performance
```

for:

```text
query optimization
caching
database performance
payload optimization
connection management
```

Use:

```text
backend-queues-workers
```

for:

```text
background jobs
long-running operations
asynchronous processing
job status
worker execution
```

Use:

```text
backend-resilience
```

for:

```text
timeouts
retries
backoff
circuit breakers
dependency failures
graceful degradation
```

Use:

```text
backend-observability
```

for:

```text
logging
metrics
tracing
request IDs
health checks
monitoring
```

The skills should complement each other rather than duplicate one another.

---

# Final Principle

A good API should make the correct usage obvious.

The client should not have to guess:

```text
what endpoint to call
what method to use
what fields are required
what status means success
what an error means
whether the request can be retried
whether the operation is asynchronous
```

The API should be:

**Consistent. Predictable. Explicit. Secure. Maintainable. Evolvable.**

Do not design APIs for theoretical complexity.

Design them for the actual system.

Use simple CRUD when CRUD is enough.

Use explicit business actions when the domain requires them.

Use pagination when collections can grow.

Use asynchronous jobs when work is genuinely long-running.

Use versioning when compatibility requires it.

Use transactions and concurrency controls when correctness requires them.

Use documentation and stable contracts so the frontend and backend can evolve without constantly breaking each other.

The best API is not the one with the most endpoints, abstractions, or architectural patterns.

It is the one where another developer can look at it and immediately understand:

**"I know how this system works."**

```
```
