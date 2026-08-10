````md
---
name: secure-backend-development
description: Build secure, resilient, and production-ready backend systems with strong authentication, authorization, password protection, token security, session management, validation, database security, API protection, rate limiting, abuse prevention, secure file handling, business-logic protection, secrets management, dependency security, monitoring, and defense against current common attack patterns. Use this skill whenever designing, implementing, reviewing, refactoring, or securing backend APIs, authentication systems, databases, middleware, routes, controllers, services, or server-side business logic.
---

# Secure Backend Development

Build backend systems with security as a normal part of the architecture rather than something added after the application is already complete.

Security should be treated as a system-wide concern.

The objective is to build backends that are:

- secure
- predictable
- resilient
- maintainable
- auditable
- appropriately restrictive
- resistant to common attacks
- resistant to abuse
- safe when exposed to untrusted clients
- capable of recovering from failures
- appropriate for the project's actual risk level

Do not add security mechanisms merely because they exist.

Choose security controls according to:

- application sensitivity
- user roles
- data sensitivity
- attack surface
- deployment environment
- business requirements
- regulatory requirements where applicable
- expected traffic
- consequences of compromise

Prefer practical security that can actually be maintained.

---

# Core Security Philosophy

## Security Is a System Property

Do not assume that one security mechanism makes an application secure.

A secure backend usually requires multiple layers:

```text
Authentication
        ↓
Authorization
        ↓
Input Validation
        ↓
Business Logic Protection
        ↓
Database Protection
        ↓
Resource Protection
        ↓
Logging and Monitoring
        ↓
Recovery
````

Each layer should assume that another layer may eventually fail.

---

# Never Trust the Client

Anything coming from the client should be considered untrusted.

This includes:

* request body
* query parameters
* route parameters
* headers
* cookies
* tokens
* uploaded files
* filenames
* MIME types
* client timestamps
* client-generated IDs
* client-calculated prices
* client-calculated permissions
* client-generated status values
* client-generated ownership information
* hidden form fields
* application state
* values stored in local storage
* values stored in AsyncStorage
* values returned by previous API calls

Never assume that because the frontend normally sends a certain value, a malicious client cannot send something else.

The backend must enforce its own rules.

---

# Threat Model First

Before implementing security, identify what needs protection.

Consider:

* Who can access the system?
* What roles exist?
* What data is sensitive?
* What actions are dangerous?
* What resources are expensive?
* What can be abused?
* What happens if an account is compromised?
* What happens if an administrator account is compromised?
* What happens if a token is stolen?
* What happens if a database is leaked?
* What happens if an uploaded file is malicious?
* What happens if a request is repeated?
* What happens if requests arrive concurrently?
* What happens if a user manipulates IDs?
* What happens if a user skips the frontend completely?

Do not design security only around normal user behavior.

Design around hostile behavior as well.

---

# Defense in Depth

Do not depend on one security mechanism.

For sensitive operations, use multiple appropriate controls.

For example:

```text
Authenticated user
+
Correct role
+
Resource ownership
+
Input validation
+
Business rule validation
+
Rate limiting
+
Audit logging
```

The exact combination depends on the operation.

Do not blindly add every control to every endpoint.

---

# Principle of Least Privilege

Give users, services, database accounts, and processes only the permissions they need.

Avoid:

```text
every authenticated user
→ access everything
```

Prefer:

```text
user
→ only permitted resources

role
→ only permitted actions

service
→ only required infrastructure permissions
```

Apply least privilege to:

* users
* roles
* database users
* API keys
* service accounts
* cloud resources
* file storage
* background workers
* administrative functions

---

# Authentication

Authentication answers:

> Who is this user?

Authorization answers:

> What is this user allowed to do?

Do not treat authentication as authorization.

---

# Authentication Requirements

Authentication systems should consider:

* secure password storage
* login protection
* account enumeration
* session management
* token expiration
* refresh token security
* logout
* account recovery
* email verification
* MFA
* suspicious login detection
* reauthentication
* session revocation

The appropriate controls depend on the application's sensitivity.

---

# Password Storage

Never store passwords in plaintext.

Never store passwords using reversible encryption.

Never implement custom password hashing.

Use a password hashing algorithm designed specifically for password storage.

Appropriate choices may include:

* Argon2id
* bcrypt
* scrypt

Use the library's current recommended configuration.

---

# Bcrypt

When bcrypt is used:

* hash passwords before storing them
* never store the original password
* never log passwords
* never return password hashes to clients
* use an appropriate cost factor
* use a maintained implementation
* compare passwords using the library's verification function

Do not manually compare password hashes if the library provides a safe comparison method.

---

# Password Policies

Password requirements should balance security and usability.

Consider:

* minimum length
* breached-password detection
* password reuse
* common-password prevention
* reset behavior

Avoid unnecessary rules that make passwords harder to remember without meaningfully improving security.

Do not require arbitrary complexity rules merely because they look secure.

Length is generally more useful than excessive composition rules.

---

# Password Reset

Password reset flows are security-sensitive.

A reset system should:

* use unpredictable reset tokens
* expire reset tokens
* make reset tokens single-use
* avoid exposing whether an account exists
* invalidate the token after successful use
* avoid logging reset tokens
* avoid placing sensitive reset tokens in unnecessary locations
* consider revoking active sessions after a password reset

Do not implement password resets using predictable IDs, timestamps, usernames, or other guessable values.

---

# Password Change

Changing a password should:

* require the current password when appropriate
* require authentication
* validate the new password
* hash the new password
* avoid returning sensitive credentials
* consider revoking existing sessions
* consider requiring reauthentication for high-risk situations

---

# Account Lockout

Avoid simplistic permanent account lockouts that can be abused for denial of service.

Prefer controls such as:

* progressive delays
* login throttling
* rate limiting
* suspicious-login detection
* temporary restrictions
* additional verification

Use account lockout carefully.

---

# Login Protection

Protect login endpoints against:

* brute force
* credential stuffing
* password spraying
* automated attacks
* enumeration
* excessive requests

Consider:

* rate limits
* progressive delays
* IP-based controls
* account-based controls
* device/risk signals
* MFA
* suspicious login detection

Do not rely exclusively on IP addresses because multiple users can share an IP.

---

# Account Enumeration

Do not unnecessarily reveal whether an account exists.

Avoid responses such as:

```text
User does not exist.
```

for authentication and password recovery flows when this would allow attackers to enumerate users.

Use consistent responses where appropriate.

---

# Multi-Factor Authentication

MFA should be considered for:

* administrators
* privileged accounts
* sensitive applications
* financial operations
* security-sensitive actions

Possible mechanisms include:

* authenticator applications
* passkeys
* hardware security keys
* other strong authentication methods

Avoid treating SMS as equivalent to stronger phishing-resistant methods when stronger options are available.

---

# MFA Recovery

MFA recovery is itself a security-sensitive operation.

Protect recovery using:

* verified identity
* appropriate authentication
* recovery codes
* additional verification
* administrative controls where appropriate

Do not create a recovery mechanism that completely bypasses MFA security.

---

# Email Verification

Email verification can be used to confirm ownership of an email address.

Verification tokens should:

* be unpredictable
* expire
* be single-use
* be invalidated after successful verification

Do not consider an email merely syntactically valid.

---

# Trusted Devices

Trusted-device functionality should be treated as a security feature.

Do not trust a device merely because:

```text
device ID exists
```

or:

```text
client says "remember me"
```

Use secure, unpredictable device/session identifiers.

Consider:

* expiration
* revocation
* device management
* suspicious activity
* reauthentication
* password changes
* MFA changes

---

# Reauthentication

Require fresh authentication for particularly sensitive operations when appropriate.

Examples:

* changing password
* changing email
* disabling MFA
* changing security settings
* accessing highly sensitive information
* administrative operations
* financial actions

Do not assume that an old session is always sufficient for high-risk operations.

---

# Sessions

Sessions should have clear:

* creation
* expiration
* renewal
* revocation
* logout
* inactivity behavior

Do not create sessions that remain valid indefinitely without justification.

---

# Token Security

If using tokens:

* keep them short-lived where appropriate
* protect signing secrets
* validate signatures
* validate expiration
* validate issuer when appropriate
* validate audience when appropriate
* validate token type where appropriate
* avoid accepting unexpected algorithms
* do not put sensitive information in tokens unnecessarily

Do not assume that decoding a token proves that it is authentic.

Signature verification is required.

---

# JWT

When using JWTs:

Validate all security-relevant claims that the application relies upon.

Depending on the architecture, this may include:

* signature
* expiration
* issuer
* audience
* subject
* token type
* not-before time

Do not accept arbitrary algorithms from the token header.

Configure the accepted algorithm explicitly.

---

# Access Tokens

Access tokens should generally be short-lived enough to reduce the impact of theft.

Do not place excessive sensitive data inside access tokens.

Remember:

> A signed token is not automatically encrypted.

Do not put secrets or unnecessary personal information inside a token merely because the token is signed.

---

# Refresh Tokens

Refresh tokens require stronger protection than short-lived access tokens.

Consider:

* secure storage
* rotation
* expiration
* revocation
* reuse detection
* device/session association

Do not allow stolen refresh tokens to remain permanently valid.

---

# Refresh Token Rotation

For systems using refresh tokens, consider rotating them when they are used.

A suspicious reuse of an already-rotated refresh token may indicate token theft.

When appropriate, revoke the associated token family/session.

---

# Logout

Logout should invalidate the appropriate authentication state.

Depending on the architecture, this may include:

* session revocation
* refresh token revocation
* token family invalidation
* cookie invalidation
* device/session removal

Do not assume deleting a frontend token is equivalent to server-side revocation.

---

# Authorization

Authorization determines what an authenticated user is allowed to do.

Every protected endpoint should enforce authorization server-side.

Do not rely on:

```text
hidden frontend buttons
```

or:

```text
disabled frontend controls
```

as security mechanisms.

---

# Role-Based Access Control

For role-based systems, define explicit permissions.

For example:

```text
SUPER_ADMIN
ADMIN
STAFF
VIEWER
```

Do not scatter role checks inconsistently throughout controllers.

Prefer centralized or reusable authorization middleware where appropriate.

---

# Permission-Based Authorization

Roles can become difficult to manage in large systems.

For complex applications, consider permissions such as:

```text
drivers.read
drivers.create
drivers.update
drivers.delete
violations.read
violations.create
violations.update
reports.export
users.manage
```

Use roles as collections of permissions when appropriate.

---

# Resource Ownership

Do not assume that authentication grants access to every resource.

For example:

```text
GET /users/123
```

must verify whether the current user is actually allowed to access user `123`.

Prevent:

* IDOR
* BOLA
* unauthorized record access
* cross-user data access

---

# Object-Level Authorization

Always verify authorization against the actual resource.

Bad pattern:

```text
user is authenticated
→ allow access to any ID
```

Better:

```text
user is authenticated
+
user has permission
+
resource is within user's permitted scope
→ allow
```

---

# Function-Level Authorization

Protect sensitive functions as well as resources.

Examples:

* deleting records
* exporting records
* changing roles
* approving transactions
* changing system settings
* uploading official documents
* viewing sensitive reports

Do not assume that a hidden route is protected simply because users normally cannot navigate to it.

---

# Input Validation

Validate every untrusted input.

Validate:

* body
* query
* params
* headers where relevant
* files
* cookies
* token claims
* webhook payloads

Validation should happen server-side.

---

# Allowlist Validation

Prefer explicit allowed values where possible.

For example:

```text
status ∈ {
  REGISTERED,
  UNREGISTERED,
  COLORUM,
  IMPOUNDED,
  INACTIVE
}
```

Do not accept arbitrary strings when only a known set of values is valid.

---

# Type Validation

Validate:

* strings
* numbers
* booleans
* arrays
* objects
* dates
* IDs
* enums

Do not assume a value is the correct type because the frontend normally sends it that way.

---

# Length Validation

Set appropriate limits for:

* usernames
* names
* descriptions
* search queries
* comments
* filenames
* uploaded metadata

This helps prevent abuse and unexpected resource consumption.

---

# Numeric Validation

Validate:

* minimum
* maximum
* integer requirements
* decimal precision
* finite values

Do not accept:

```text
NaN
Infinity
-Infinity
```

or unexpectedly large numeric values where they are not valid.

---

# Date Validation

Validate dates server-side.

Consider:

* valid date format
* timezone behavior
* minimum/maximum dates
* impossible dates
* future dates
* historical constraints

Do not trust client timestamps for security decisions without appropriate server-side verification.

---

# Output Validation

Do not only validate input.

Validate important data before returning or processing it.

Avoid accidentally returning:

* password hashes
* refresh tokens
* internal secrets
* private keys
* internal stack traces
* sensitive database fields
* internal infrastructure details

---

# Mass Assignment

Do not allow clients to update arbitrary model fields.

Bad:

```text
Model.update(req.body)
```

when the request body can contain privileged fields.

Prefer explicit allowed fields.

For example:

```text
allowed:
name
email
phone
```

while preventing:

```text
role
status
permissions
createdBy
verified
```

from being changed unless explicitly authorized.

---

# Database Security

The database should be treated as a separate security boundary.

Protect:

* credentials
* network access
* database permissions
* indexes
* query construction
* backups
* logs
* sensitive fields

---

# Database Credentials

Never hardcode database credentials in source code.

Use:

* environment variables
* secret managers
* deployment secret storage

Do not commit credentials to Git.

---

# Database Least Privilege

The application database user should only have the permissions it requires.

Do not use unrestricted administrative database credentials for the application when a restricted account is sufficient.

---

# NoSQL Injection

NoSQL databases are not automatically safe from injection.

Validate query input.

Do not blindly pass client objects into database filters.

Be especially careful with operators such as:

```text
$gt
$gte
$lt
$lte
$ne
$in
$nin
$regex
$where
```

depending on the database and driver.

---

# SQL Injection

When using SQL:

* use parameterized queries
* use prepared statements
* use safe query builders/ORMs appropriately

Never concatenate untrusted user input into SQL statements.

---

# Query Limits

Protect expensive database operations.

Consider limits for:

* result size
* page size
* sorting fields
* filtering complexity
* aggregation complexity
* search complexity

Do not allow arbitrary client-controlled queries to consume unlimited database resources.

---

# Pagination

Use pagination for large datasets.

Do not return thousands or millions of records simply because a client requested them.

Set reasonable maximum page sizes.

---

# Sorting

If clients can request sorting, use an allowlist of permitted fields.

Do not blindly pass arbitrary client-provided field names into database queries.

---

# Search

Search endpoints should be protected against:

* excessive query length
* expensive regular expressions
* wildcard abuse
* repeated requests
* expensive database scans

Use:

* indexes
* query limits
* validation
* rate limiting
* controlled search syntax

---

# API Security

Every API endpoint should have a clear security classification.

For example:

```text
Public
Authenticated
Role-protected
Permission-protected
Administrative
Internal
Webhook
System-only
```

Do not leave endpoint security ambiguous.

---

# Route Organization

Separate responsibilities clearly.

A practical backend structure may include:

```text
src/
├── config/
├── controllers/
├── middleware/
├── models/
├── routes/
├── services/
├── validators/
├── utils/
├── workers/
├── jobs/
└── app.js
```

The exact structure may vary by project size.

Do not create files merely to make the folder tree look professional.

---

# Controllers

Controllers should primarily coordinate:

```text
request
→ validation
→ service
→ response
```

Avoid putting the entire business logic inside route handlers.

---

# Services

Services should contain reusable business logic where appropriate.

This makes it easier to:

* test
* reuse
* secure
* maintain
* reason about business rules

Do not create a service layer for every trivial function if it adds unnecessary complexity.

---

# Middleware

Middleware is appropriate for cross-cutting concerns such as:

* authentication
* authorization
* rate limiting
* request IDs
* validation
* logging
* security headers

Be careful about middleware scope.

---

# Middleware Scope

Avoid accidentally applying security middleware to unrelated routes.

Prefer:

```text
app.use("/api/drivers", driverRoutes)
app.use("/api/enforcers", enforcerRoutes)
```

with route-specific middleware inside the appropriate router.

For example:

```text
driverRoutes
→ authentication
→ driver authorization
→ driver controller

enforcerRoutes
→ authentication
→ enforcer authorization
→ enforcer controller
```

Do not attach middleware globally unless it is genuinely intended to affect every route.

---

# Global vs Route-Specific Security

Global middleware is appropriate for controls such as:

* request parsing
* security headers
* request IDs
* general logging
* global rate limiting where appropriate

Route-specific middleware is appropriate for:

* role checks
* permission checks
* resource-specific authorization
* special validation
* sensitive operations

Always verify middleware ordering.

---

# Middleware Ordering

Middleware order can affect security.

For example:

```text
request
→ security headers
→ request ID
→ parsing
→ authentication
→ authorization
→ validation
→ rate limit
→ controller
→ error handler
```

The correct order depends on the application.

Do not assume middleware order is interchangeable.

---

# Error Handling

Do not expose internal implementation details to clients.

Avoid returning:

* stack traces
* database errors
* filesystem paths
* secret values
* internal service URLs
* SQL queries
* internal object structures

Return appropriate client-facing errors.

---

# Error Consistency

Use consistent API error structures.

For example:

```json
{
  "success": false,
  "message": "Access denied",
  "code": "FORBIDDEN"
}
```

The exact format depends on the project.

Avoid leaking security-sensitive information through differences in error messages.

---

# HTTP Status Codes

Use status codes appropriately.

Examples:

```text
200 OK
201 Created
204 No Content
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Content
429 Too Many Requests
500 Internal Server Error
```

Do not use `200` for every failure.

---

# Security Headers

Use appropriate security-related HTTP headers.

Depending on the application, consider:

* Content-Security-Policy
* Strict-Transport-Security
* X-Content-Type-Options
* Referrer-Policy
* Permissions-Policy
* appropriate frame protections

Do not blindly copy a header configuration without understanding the application.

---

# HTTPS

Sensitive traffic should use HTTPS.

Do not send:

* passwords
* session tokens
* authentication credentials
* sensitive personal information

over plaintext HTTP in production.

---

# Cookies

If using authentication cookies, consider:

* `HttpOnly`
* `Secure`
* appropriate `SameSite`
* appropriate expiration
* narrow path/domain scope

Do not store sensitive authentication information in insecure browser-accessible storage when a safer architecture is available.

---

# CORS

Configure CORS deliberately.

Do not use:

```text
Access-Control-Allow-Origin: *
```

for sensitive authenticated applications without understanding the consequences.

Allow only appropriate origins.

Do not assume CORS is an authentication mechanism.

CORS controls browser behavior.

It does not prevent direct API requests by attackers.

---

# CSRF

If authentication relies on cookies, evaluate CSRF protection.

Possible controls include:

* SameSite cookies
* CSRF tokens
* origin validation
* appropriate request design

Do not assume that CORS alone eliminates CSRF.

---

# Rate Limiting

Rate limiting protects resources from excessive use.

Apply it according to endpoint risk.

Sensitive endpoints often deserve stricter limits:

* login
* password reset
* verification
* MFA
* token refresh
* expensive searches
* exports
* file uploads
* public endpoints

Do not necessarily apply the same limit to every endpoint.

---

# Rate Limit Dimensions

Rate limits can consider:

* IP
* authenticated user
* API key
* session
* endpoint
* resource
* device
* account

Avoid relying exclusively on one dimension.

---

# Distributed Rate Limiting

If the backend runs across multiple instances, in-memory rate limiting may not provide consistent protection.

Consider shared infrastructure such as:

* Redis
* gateway-level rate limiting
* load-balancer controls
* managed infrastructure controls

Choose according to deployment architecture.

---

# Brute Force Protection

For authentication endpoints, combine:

* rate limiting
* credential protection
* suspicious activity detection
* MFA where appropriate
* progressive delays

Avoid relying solely on account lockout.

---

# Resource Exhaustion

Security includes protecting server resources.

Consider limits on:

* request body size
* file size
* number of uploaded files
* pagination size
* query complexity
* search complexity
* concurrent operations
* expensive jobs
* request duration

---

# File Upload Security

Treat every uploaded file as untrusted.

Validate:

* size
* type
* extension
* actual file signature where appropriate
* filename
* storage location

Do not trust the client-provided MIME type alone.

---

# File Names

Do not directly use user-provided filenames as filesystem paths.

Prevent:

* path traversal
* overwriting
* special-character problems
* executable filename attacks

Generate safe server-side filenames or storage keys.

---

# File Storage

Store uploaded files outside sensitive executable directories where possible.

For production systems, consider object storage such as a secure cloud storage service.

Restrict access to uploaded files appropriately.

---

# Image Uploads

Images are still untrusted files.

Consider:

* file signature validation
* size limits
* image decoding safety
* metadata handling
* malware scanning where appropriate
* re-encoding when appropriate

Do not assume:

```text
.jpg
```

means the file is actually a safe image.

---

# Document Uploads

Documents such as:

* PDFs
* receipts
* scanned forms
* office files

should be treated as untrusted.

Consider:

* file type verification
* size limits
* malware scanning
* safe storage
* controlled downloads
* authorization
* filename sanitization

---

# Cloud Storage

If using services such as object storage or media storage:

* keep credentials server-side
* use scoped credentials
* avoid public access unless required
* use signed URLs when appropriate
* control expiration
* validate uploaded content
* avoid exposing internal storage identifiers unnecessarily

---

# Path Traversal

Never construct filesystem paths directly from untrusted input.

Reject or safely normalize dangerous path patterns.

Do not allow clients to determine arbitrary filesystem locations.

---

# Webhooks

Webhook endpoints should not automatically trust incoming requests.

Verify:

* signatures
* secrets
* timestamps
* replay protection where appropriate
* payload integrity

Do not process sensitive webhook actions before verification.

---

# Replay Protection

For sensitive signed requests, consider whether an attacker could capture and replay a valid request.

Possible protections include:

* timestamps
* nonces
* request IDs
* short validity windows
* idempotency keys

---

# Idempotency

Sensitive operations may need idempotency.

Examples:

* payments
* transaction creation
* ticket creation
* external API operations
* email sending
* document processing

The same request should not accidentally create multiple results when the operation is intended to happen once.

---

# Race Conditions

Security rules can fail when operations occur concurrently.

Examples:

```text
request A checks balance
request B checks balance
request A updates
request B updates
```

Use appropriate:

* atomic database operations
* transactions
* unique constraints
* locks
* optimistic concurrency
* idempotency

depending on the problem.

---

# Unique Constraints

Do not rely only on:

```text
check if username exists
→ create username
```

Two concurrent requests can pass the check.

Use database-level unique constraints where appropriate.

---

# State Transitions

Protect business-critical status transitions.

For example:

```text
REGISTERED
→ INACTIVE
```

may be valid.

But:

```text
INACTIVE
→ REGISTERED
```

may require specific authorization.

Define allowed transitions explicitly.

Do not allow clients to directly assign arbitrary status values.

---

# Business Logic Security

Security vulnerabilities are not limited to technical bugs.

Business logic must be protected.

Examples:

* users modifying records they do not own
* users skipping approval steps
* users changing prices
* users changing transaction totals
* users assigning themselves privileged roles
* users bypassing required verification
* users repeating one-time actions
* users manipulating status transitions

Always validate business rules on the backend.

---

# Never Trust Client Calculations

Do not trust the frontend for authoritative values such as:

* prices
* fares
* discounts
* totals
* permissions
* roles
* ownership
* approval state

The backend should recalculate or verify values that affect security or business integrity.

---

# Audit Logging

Sensitive operations should be auditable.

Consider logging events such as:

* login
* failed login
* logout
* password change
* password reset
* MFA changes
* role changes
* permission changes
* record deletion
* sensitive exports
* administrative changes
* security setting changes

Do not log sensitive secrets.

---

# Security Logs

Security logs should help answer:

```text
Who?
What?
When?
From where?
To which resource?
What happened?
Was it successful?
```

Use request IDs or correlation IDs where appropriate.

---

# Do Not Log Secrets

Never log:

* passwords
* password reset tokens
* access tokens
* refresh tokens
* API secrets
* private keys
* database passwords
* full authentication headers

Be careful with request bodies because they may contain credentials or personal data.

---

# Personal Data

Minimize collection of sensitive data.

Ask:

> Does the system actually need this data?

If not, do not collect it.

Protect data that is collected.

---

# Data Exposure

Return only the fields the client needs.

Avoid:

```text
return entire database document
```

when the document contains internal or sensitive fields.

Prefer explicit response objects.

---

# Sensitive Exports

Exports can expose large amounts of data.

Protect:

* Excel exports
* CSV exports
* PDF reports
* administrative reports
* bulk downloads

Consider:

* authorization
* rate limiting
* audit logging
* filtering
* data minimization

---

# Secrets Management

Never commit secrets into source control.

Protect:

* database credentials
* JWT signing secrets
* API keys
* cloud credentials
* SMTP credentials
* encryption keys
* webhook secrets

Use environment variables or dedicated secret management systems.

---

# Environment Variables

Validate required environment variables during startup.

Fail safely if critical secrets are missing.

Do not silently use insecure production defaults.

---

# Secret Rotation

Design important credentials so they can be rotated.

Consider rotation for:

* API keys
* JWT signing keys
* database credentials
* webhook secrets
* cloud credentials

Do not hardcode secrets into application logic.

---

# Encryption

Use encryption where appropriate.

Protect sensitive data:

* in transit
* at rest where required
* during storage
* during transfer

Do not invent custom encryption algorithms.

Use well-maintained cryptographic libraries.

---

# Cryptography

Never implement custom cryptographic primitives.

Use established algorithms and libraries.

Avoid:

* custom hashing
* homemade encryption
* predictable tokens
* weak random number generation

Use cryptographically secure randomness for security-sensitive values.

---

# Dependency Security

Keep dependencies maintained.

Regularly review:

* outdated packages
* known vulnerabilities
* abandoned packages
* unnecessary packages
* transitive dependencies

Use automated dependency scanning where appropriate.

---

# Supply Chain Security

Be careful when installing packages.

Evaluate:

* package maintenance
* popularity and ecosystem trust
* release history
* permissions
* suspicious behavior
* dependency changes

Do not install a package merely because it solves a tiny problem.

---

# API Documentation

Maintain an accurate API contract.

Document:

* authentication
* authorization
* request format
* response format
* validation
* errors
* rate limits
* pagination
* security requirements

An undocumented API is harder to secure consistently.

---

# API Versioning

For larger systems, consider versioning APIs.

For example:

```text
/api/v1/...
```

Do not break existing clients unexpectedly.

Security-sensitive changes should be documented.

---

# Health Endpoints

Health endpoints should reveal only what is necessary.

Avoid exposing:

* database credentials
* internal configuration
* stack traces
* detailed infrastructure information

A health endpoint should confirm service health without becoming an information leak.

---

# Internal Endpoints

Internal endpoints should not automatically be considered safe.

If an endpoint can be reached by an attacker, treat it as exposed.

Protect internal functionality using:

* authentication
* network controls
* service authentication
* authorization

where appropriate.

---

# Administrative APIs

Administrative endpoints deserve stronger protection.

Consider:

* MFA
* role/permission checks
* reauthentication
* rate limiting
* audit logs
* IP/network restrictions where appropriate
* session controls

Do not rely solely on an `/admin` URL prefix.

---

# Database Backups

Backups contain the same sensitive information as the production database.

Protect:

* access
* encryption
* retention
* deletion
* restoration procedures

Do not leave backups publicly accessible.

---

# Backup Testing

A backup that cannot be restored is not a reliable backup.

Periodically test:

* restoration
* integrity
* access
* recovery procedures

---

# Incident Preparedness

Prepare for compromise rather than assuming compromise is impossible.

Know how to:

* revoke sessions
* rotate secrets
* disable compromised accounts
* block abusive clients
* restore backups
* investigate logs
* deploy emergency patches

---

# Security Monitoring

Monitor important security signals.

Examples:

* repeated failed logins
* unusual authentication activity
* token reuse
* privilege changes
* large exports
* unusual upload activity
* rate-limit violations
* repeated authorization failures
* unexpected error spikes

---

# Alerting

Alerts should focus on actionable events.

Avoid alerting on everything.

Good alerts should answer:

```text
What happened?
How serious is it?
What should be done?
```

---

# Current Attack Awareness

Security practices should not remain static.

When working on a significant or security-sensitive project, consider current common attack trends and relevant vulnerability classes.

Review relevant sources such as:

* OWASP guidance
* known vulnerability databases
* framework security advisories
* dependency advisories
* cloud provider security guidance

Do not blindly implement a trend simply because it is new.

Determine whether it applies to the actual architecture.

---

# OWASP Awareness

Use OWASP guidance as a major reference for web application security.

Pay particular attention to categories such as:

* broken access control
* cryptographic failures
* injection
* insecure design
* security misconfiguration
* vulnerable components
* identification and authentication failures
* software and data integrity failures
* security logging and monitoring failures
* server-side request forgery where relevant

Do not treat a checklist as proof of security.

---

# SSRF

If the backend can request external URLs based on user input, evaluate SSRF risks.

Do not blindly allow clients to control server-side network requests.

Protect against access to:

* internal services
* localhost
* cloud metadata services
* private network ranges
* internal administration endpoints

Use allowlists where practical.

---

# Injection

Consider injection risks across all interpreters and systems.

Potential targets include:

* SQL
* NoSQL
* shell commands
* templates
* LDAP
* file paths
* HTML
* regular expressions
* external service queries

Do not assume an ORM or framework automatically prevents every injection class.

---

# Command Execution

Never pass untrusted input directly into operating-system commands.

If command execution is genuinely required:

* use safe APIs
* use strict allowlists
* avoid shell interpretation
* validate arguments
* restrict privileges

---

# Regular Expression Abuse

User-controlled regular expressions can cause excessive CPU consumption.

Be cautious with:

* user-provided regex
* complex patterns
* catastrophic backtracking
* unrestricted search expressions

Prefer safer search mechanisms when possible.

---

# Denial of Service

Security includes availability.

Protect against:

* excessive requests
* expensive database queries
* large payloads
* large files
* recursive processing
* expensive regex
* unbounded loops
* huge exports
* excessive concurrent jobs

---

# Queues and Background Work

Move expensive or non-immediate work out of the request-response path when appropriate.

Examples:

* email
* document processing
* report generation
* image processing
* notifications
* large exports
* external API synchronization

Use background workers when they improve responsiveness and reliability.

---

# Worker Security

Workers should have only the permissions they need.

Protect jobs against:

* duplicate processing
* malformed payloads
* poisoned data
* infinite retries
* unbounded queue growth

Validate job payloads just like HTTP requests.

---

# Retry Safety

Retries can accidentally repeat operations.

Before retrying a job or request, determine whether the operation is idempotent.

Do not blindly retry:

```text
create transaction
```

without ensuring duplicate creation cannot occur.

---

# Transactions

Use database transactions when multiple changes must succeed or fail together.

Examples:

```text
create transaction
+
update balance
+
create audit record
```

If these operations must remain consistent, use appropriate transactional mechanisms.

---

# Concurrency

Security and correctness can fail under concurrent requests.

Protect critical operations using appropriate:

* atomic updates
* transactions
* unique indexes
* optimistic locking
* pessimistic locking
* idempotency

Choose based on the actual consistency requirement.

---

# API Responsiveness

A secure backend should also remain responsive.

Avoid performing unnecessarily expensive operations synchronously during normal API requests.

Consider:

* caching
* pagination
* queues
* workers
* efficient queries
* connection pooling
* request timeouts
* cancellation
* streaming where appropriate

Security controls should not accidentally make normal endpoints unusably slow.

---

# Request Timeouts

External requests should have appropriate timeouts.

Do not allow a backend request to wait indefinitely for:

* external APIs
* databases
* file services
* network resources

Unbounded waiting can consume server resources.

---

# External Services

Treat external APIs as untrusted dependencies.

Validate:

* response format
* status
* size
* timeout
* expected fields

Do not assume external services will always respond correctly.

---

# Caching

Caching can improve performance but can also introduce security problems.

Do not cache sensitive information where another user could receive it.

Consider:

* user-specific cache keys
* authorization boundaries
* expiration
* invalidation
* sensitive responses

---

# Authorization and Caching

Always consider:

```text
Can cached data cross user boundaries?
```

A cache bug can become a data exposure vulnerability.

---

# Response Compression

Compression can improve performance but may introduce risks in certain contexts.

Evaluate compression carefully when responses contain secrets mixed with attacker-controlled content.

Do not enable advanced performance features without understanding their security implications.

---

# Dependency and Framework Updates

Keep the backend framework and security libraries updated.

When updating:

* review changelogs
* review security advisories
* test authentication
* test authorization
* test middleware
* test validation
* test database behavior

Do not blindly update production dependencies without testing.

---

# Testing Security

Security controls should be tested.

Test:

* unauthenticated requests
* unauthorized requests
* malformed input
* invalid tokens
* expired tokens
* revoked tokens
* invalid roles
* invalid IDs
* cross-user access
* duplicate requests
* concurrent requests
* oversized requests
* rate limits
* upload restrictions
* password reset
* MFA
* session revocation

---

# Negative Testing

Do not only test:

```text
valid request
→ success
```

Also test:

```text
invalid request
→ rejected
```

and:

```text
malicious request
→ safely rejected
```

---

# Authorization Testing

For every protected endpoint, test at least:

```text
no authentication
→ rejected

authenticated but wrong role
→ rejected

correct role but wrong resource
→ rejected

correct user and permission
→ allowed
```

---

# Security Regression Tests

When fixing a vulnerability, add a test that prevents the vulnerability from returning.

Security fixes should become part of the project's permanent test suite.

---

# Production Configuration

Development and production should not use identical security settings.

Production should:

* disable unnecessary debug output
* protect secrets
* use HTTPS
* restrict CORS
* configure secure cookies
* limit administrative access
* use production logging
* use appropriate rate limits
* protect database access

---

# Debugging

Never expose development debugging information in production.

Avoid returning:

```text
stack traces
database queries
environment variables
filesystem paths
internal configuration
```

to clients.

---

# Secure Defaults

When a developer does not explicitly configure a security option, prefer a safe default when practical.

Examples:

* deny access unless authorized
* reject unknown fields where appropriate
* limit request sizes
* expire sessions
* require authentication for sensitive routes

---

# Fail Securely

When security checks fail, the system should fail closed where appropriate.

Prefer:

```text
authorization unavailable
→ deny access
```

over:

```text
authorization unavailable
→ allow access
```

Do not let errors silently disable security controls.

---

# Do Not Hide Security Failures

Security failures should be observable to the system operators.

Do not silently swallow:

* authorization errors
* token validation failures
* suspicious activity
* failed verification
* security middleware errors

Handle them safely while preserving useful internal observability.

---

# Security Boundaries

Clearly identify boundaries between:

```text
Client
↓
API
↓
Application
↓
Database
↓
External Services
↓
Workers
↓
Storage
```

Each boundary should validate assumptions.

---

# Backend Structure

A scalable backend may use:

```text
src/
├── config/
│   ├── database.js
│   ├── security.js
│   └── environment.js
│
├── controllers/
│
├── middleware/
│   ├── auth.js
│   ├── authorization.js
│   ├── validation.js
│   ├── rateLimit.js
│   └── errorHandler.js
│
├── models/
│
├── routes/
│
├── services/
│
├── validators/
│
├── workers/
│
├── jobs/
│
├── utils/
│
└── app.js
```

This is a guideline, not a requirement.

Do not create unnecessary folders simply to imitate a large enterprise architecture.

---

# Controller and Route Separation

Separating routes and controllers is often useful when the application grows.

For example:

```text
routes/
→ endpoint definitions

controllers/
→ request/response coordination

services/
→ business logic

models/
→ persistence
```

This can improve maintainability.

However, a small endpoint does not need to be split across many files if doing so makes the project harder to understand.

Use structure according to complexity.

---

# Security Middleware Organization

Security middleware should have clear scope.

For example:

```text
global middleware
→ applies to the entire application

router middleware
→ applies to one feature

route middleware
→ applies to one endpoint

controller/service validation
→ applies to a specific business rule
```

Do not accidentally make a security control global when it was intended for one route.

---

# Authentication Architecture

Keep authentication responsibilities clearly separated.

A typical flow may be:

```text
Request
↓
Authentication middleware
↓
Token/session validation
↓
Authenticated identity
↓
Authorization middleware
↓
Resource authorization
↓
Controller
↓
Service
```

Do not mix unrelated authentication logic into every controller.

---

# Security Checklist

Before considering a backend production-ready, verify:

## Authentication

* [ ] Passwords are securely hashed
* [ ] Password hashes are never returned
* [ ] Login is protected against brute force
* [ ] Password reset tokens are secure
* [ ] Password reset tokens expire
* [ ] Password reset tokens are single-use
* [ ] Email verification is protected
* [ ] MFA is considered where appropriate
* [ ] Sessions expire appropriately
* [ ] Sessions can be revoked
* [ ] Refresh tokens are protected
* [ ] Sensitive operations can require reauthentication

## Authorization

* [ ] Every sensitive route is protected
* [ ] Roles are validated server-side
* [ ] Permissions are validated server-side
* [ ] Resource ownership is checked
* [ ] Administrative actions are protected
* [ ] Hidden frontend controls are not treated as security

## Input

* [ ] Request bodies are validated
* [ ] Query parameters are validated
* [ ] Route parameters are validated
* [ ] File uploads are validated
* [ ] Length limits exist
* [ ] Numeric limits exist
* [ ] Enum values are restricted
* [ ] Mass assignment is prevented

## Database

* [ ] Credentials are not hardcoded
* [ ] Database permissions follow least privilege
* [ ] Queries are protected against injection
* [ ] Large queries are limited
* [ ] Pagination is implemented where needed
* [ ] Sensitive fields are not exposed
* [ ] Unique constraints protect critical identifiers

## API

* [ ] HTTPS is used in production
* [ ] CORS is configured intentionally
* [ ] Authentication is enforced where required
* [ ] Authorization is enforced where required
* [ ] Rate limits exist where appropriate
* [ ] Request sizes are limited
* [ ] Errors do not expose internals
* [ ] Sensitive endpoints are protected

## Files

* [ ] Upload size is limited
* [ ] File type is validated
* [ ] MIME type is not blindly trusted
* [ ] Filenames are sanitized
* [ ] Path traversal is prevented
* [ ] Files are stored safely
* [ ] Downloads are authorized
* [ ] Malware scanning is considered for high-risk uploads

## Business Logic

* [ ] Client calculations are not trusted
* [ ] Status transitions are validated
* [ ] Duplicate operations are prevented
* [ ] Race conditions are considered
* [ ] Critical operations are idempotent where necessary
* [ ] Transactions are used where required

## Secrets

* [ ] Secrets are not committed
* [ ] Environment variables are protected
* [ ] Production secrets are managed securely
* [ ] Secrets can be rotated
* [ ] Tokens are not logged

## Monitoring

* [ ] Security events are logged
* [ ] Authentication failures are observable
* [ ] Authorization failures are observable
* [ ] Sensitive administrative actions are audited
* [ ] Alerts exist for important security events
* [ ] Logs do not contain secrets

## Resilience

* [ ] External requests have timeouts
* [ ] Expensive work can be moved to workers
* [ ] Queue jobs are retry-safe
* [ ] Large operations are controlled
* [ ] Database operations are bounded
* [ ] Rate limiting protects expensive endpoints
* [ ] Backups exist
* [ ] Backups can be restored

---

# Final Security Review

Before shipping, ask:

```text
Can an unauthenticated user access sensitive data?
```

```text
Can an authenticated user access another user's data?
```

```text
Can a normal user perform an administrative action?
```

```text
Can the client change fields it should not control?
```

```text
Can an attacker brute-force authentication?
```

```text
Can a stolen token remain valid indefinitely?
```

```text
Can the same sensitive operation be repeated accidentally?
```

```text
Can concurrent requests bypass a business rule?
```

```text
Can an uploaded file escape its intended storage boundary?
```

```text
Can a malicious request consume unreasonable server resources?
```

```text
Can an attacker discover whether accounts exist?
```

```text
Can sensitive information appear in logs?
```

```text
Can a compromised account escalate its privileges?
```

```text
Can the application recover if credentials are compromised?
```

If the answer to any of these reveals a meaningful vulnerability, address it before considering the system secure.

---

# Final Principles

Security should be:

**Designed in.**

Not:

**Added later.**

Never trust the client.

Never store plaintext passwords.

Never assume authentication means authorization.

Never assume hidden frontend controls are security.

Never expose secrets through responses or logs.

Never trust user-controlled IDs without resource authorization.

Never trust uploaded files.

Never trust client-calculated security or business values.

Never allow unbounded resource consumption.

Never assume concurrency cannot happen.

Never assume a framework automatically solves every security problem.

Use established security libraries and standards.

Keep dependencies updated.

Monitor important security events.

Test security controls.

Keep security proportional to the actual project.

The final backend should be:

**Secure. Resilient. Restrictive where necessary. Observable. Maintainable.**

Security should feel like a normal part of the backend architecture, not a collection of emergency patches added after the application is finished.

```
```
