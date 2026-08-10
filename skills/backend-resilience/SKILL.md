````md
---
name: backend-rate-limiting
description: Design and implement robust backend rate limiting to protect APIs, authentication systems, databases, queues, external services, and other resources from abuse, accidental request floods, brute-force attacks, denial-of-service conditions, and excessive client usage. Use this skill when building or reviewing backend APIs where request frequency, resource consumption, authentication attempts, concurrency, or external service limits need to be controlled. Apply rate limiting according to endpoint risk and resource cost rather than blindly applying one global limit to every route.
---

# Backend Rate Limiting

Design backend rate limiting as a protection and resource-management mechanism rather than simply a request counter.

The goal is to ensure that:

- legitimate users can use the system normally
- abusive clients cannot consume unlimited resources
- expensive endpoints receive stronger protection
- authentication endpoints are protected against brute-force attacks
- databases and external services are not overwhelmed
- traffic spikes can be absorbed safely
- rate limits are predictable
- clients receive useful feedback when limited
- trusted internal operations can have appropriate limits
- rate limiting does not accidentally become a usability problem

Do not apply one arbitrary limit to every endpoint.

A:

```text
GET /places
````

endpoint and:

```text
POST /auth/login
```

endpoint do not have the same security or resource requirements.

Likewise:

```text
GET /drivers
```

and:

```text
POST /reports/generate
```

may have completely different computational costs.

Rate limits should reflect the endpoint's:

* risk
* cost
* expected usage
* user behavior
* authentication state
* resource consumption
* business requirements

---

# Core Principles

## Rate Limiting Is Not Only Security

Rate limiting protects against more than malicious attackers.

It also protects against:

* accidental request loops
* frontend bugs
* duplicated requests
* aggressive polling
* misconfigured clients
* mobile network retries
* crawler activity
* compromised accounts
* API abuse
* sudden traffic spikes

For example:

```text
Frontend bug
    ↓
Request fires repeatedly
    ↓
500 requests/second
    ↓
Database overloaded
```

Rate limiting can prevent one faulty client from damaging the entire system.

---

# Rate Limit by Resource

Think about what you are protecting.

Possible resources include:

* API server
* database
* authentication service
* email provider
* SMS provider
* file storage
* external APIs
* queue
* CPU
* memory
* expensive computation
* report generation
* search service

A route may need stronger limits if it consumes significantly more resources.

---

# Do Not Use One Global Limit

Avoid blindly implementing:

```text
100 requests / minute
```

for every endpoint.

This can cause problems.

For example:

```text
GET /health
100/min
```

may be perfectly reasonable.

But:

```text
POST /auth/login
100/min
```

may be dangerously permissive.

And:

```text
POST /reports/generate
100/min
```

may be enough to overwhelm the server.

Instead, use endpoint-specific policies.

---

# Example Rate-Limit Categories

A project may use categories such as:

```text
Public API
Authenticated API
Authentication
Password operations
Verification
File uploads
Search
Expensive operations
Administrative operations
Internal services
```

Each category can have different policies.

---

# Public Endpoints

Public endpoints are especially vulnerable because attackers do not need an account.

Examples:

```text
GET /places
GET /fare-matrix
GET /metadata
POST /contact
```

Apply reasonable limits based on expected usage.

Do not make public endpoints so restrictive that normal users are constantly blocked.

---

# Authenticated Endpoints

Authenticated users can often receive different limits.

For example:

```text
anonymous client
    ↓
stricter limits

authenticated user
    ↓
higher appropriate limits
```

However, authentication does not mean unlimited access.

A compromised account could otherwise become an abuse source.

---

# Authentication Rate Limiting

Authentication endpoints require particularly careful rate limiting.

Examples:

```text
POST /auth/login
POST /auth/register
POST /auth/forgot-password
POST /auth/reset-password
POST /auth/verify-email
POST /auth/resend-verification
POST /auth/2fa/verify
POST /auth/2fa/resend
```

These endpoints can be targeted for:

* brute-force attacks
* credential stuffing
* account enumeration
* OTP abuse
* email bombing
* SMS abuse
* password reset abuse

Use stronger limits than ordinary API endpoints.

---

# Login Rate Limiting

Do not rely solely on IP-based limits.

Attackers can distribute requests across many IP addresses.

Where appropriate, consider multiple dimensions:

```text
IP address
+
username/account
+
device/session
+
network
```

For example:

```text
IP limit
```

protects the infrastructure.

While:

```text
account-specific limit
```

protects an individual account.

---

# Account-Based Limits

A login policy may conceptually look like:

```text
Per IP:
limited attempts

Per account:
limited attempts

Global:
overall authentication traffic limit
```

This creates multiple layers of protection.

Do not reveal whether a username exists merely because different rate-limit behavior occurs.

---

# Credential Stuffing

Attackers may attempt:

```text
username1 + password1
username2 + password2
username3 + password3
...
```

across many accounts.

A single IP limit may not stop this.

Use layered controls such as:

* IP-based limits
* account-based limits
* suspicious login detection
* progressive delays
* breached-password detection where appropriate
* MFA
* device or risk signals

Do not assume rate limiting alone solves credential stuffing.

---

# Password Reset Rate Limiting

Password reset endpoints can be abused to:

* spam users
* trigger excessive emails
* enumerate accounts
* consume email provider quotas

Apply rate limits to:

```text
reset requests
verification emails
OTP requests
```

Use generic responses where necessary.

Prefer:

```text
"If the account exists, instructions will be sent."
```

rather than:

```text
"That email does not exist."
```

when account enumeration is a concern.

---

# OTP and 2FA Rate Limiting

OTP verification requires strict limits.

Protect:

```text
POST /2fa/verify
```

against repeated guesses.

Also protect:

```text
POST /2fa/resend
```

against message abuse.

Do not allow unlimited resend requests.

Use:

* attempt limits
* cooldown periods
* expiration
* attempt counters
* temporary blocking where appropriate

---

# Verification Code Expiration

Rate limiting should work together with expiration.

A verification code should not remain valid indefinitely.

Conceptually:

```text
Code generated
 ↓
Valid for limited time
 ↓
Attempt limit
 ↓
Expired
```

Do not treat rate limiting as a replacement for expiration.

---

# Sensitive Endpoint Protection

Apply stricter limits to operations such as:

```text
password change
password reset
2FA setup
2FA verification
email change
phone change
account recovery
admin login
```

These endpoints often deserve stronger protection than ordinary CRUD.

---

# Administrative Endpoints

Administrative APIs may contain sensitive operations.

Examples:

```text
DELETE /drivers/:id
POST /violations
PUT /admins/:id
POST /reports
```

Rate limiting can protect these endpoints from:

* accidental automation
* compromised administrator accounts
* malicious scripts
* excessive expensive operations

However, authorization must always remain separate.

Rate limiting does not replace:

```text
authentication
authorization
role checking
permission checking
```

---

# Rate Limiting vs Authorization

These are different controls.

Rate limiting asks:

```text
"How often can this client perform this operation?"
```

Authorization asks:

```text
"Is this client allowed to perform this operation?"
```

A user may be:

```text
authorized
```

but still:

```text
rate limited
```

Likewise, a user may be:

```text
within the rate limit
```

but:

```text
unauthorized
```

Both protections may be required.

---

# Rate Limiting vs Authentication

Authentication identifies the client.

Rate limiting controls usage.

Example:

```text
Authenticated user
        ↓
Authorization check
        ↓
Rate limit check
        ↓
Endpoint
```

The exact middleware order depends on the application's architecture and whether the limiter needs identity information.

---

# Middleware Placement

Do not automatically apply one rate limiter globally.

For example:

```js
app.use(globalLimiter);

app.use("/api/drivers", driverRoutes);
app.use("/api/enforcers", enforcerRoutes);
```

A global limiter affects all routes intentionally.

That may be useful as a broad safety layer, but it should not replace more specific limits.

Prefer a layered approach when appropriate:

```text
Global protection
        ↓
Authentication protection
        ↓
Route-specific protection
        ↓
Expensive-operation protection
```

---

# Route-Specific Middleware

Apply specialized limits to specific routes.

Conceptually:

```js
router.post(
  "/login",
  authRateLimiter,
  loginController
);
```

And:

```js
router.post(
  "/reports",
  reportRateLimiter,
  reportController
);
```

This keeps security behavior explicit.

---

# Avoid Accidental Middleware Scope

Be careful with:

```js
app.use("/api", limiter);
```

because it affects every route beneath:

```text
/api/*
```

Likewise:

```js
router.use(limiter);
```

affects all routes defined after that middleware inside the router.

Understand middleware scope before placing a limiter.

---

# Middleware Order

Middleware order matters.

Example:

```text
Request
 ↓
Security headers
 ↓
Request parsing
 ↓
Rate limiting
 ↓
Authentication
 ↓
Authorization
 ↓
Validation
 ↓
Controller
```

The correct ordering depends on what information the limiter requires.

If the limiter needs authenticated user identity, authentication may need to occur first.

Do not blindly copy middleware ordering from another project.

---

# Global Protection

A modest global rate limit can still be useful as a safety net.

Its purpose should be:

```text
protect the server from extreme request floods
```

not:

```text
define every endpoint's business usage
```

Keep global limits broad enough to avoid interfering with normal operation.

---

# Layered Rate Limiting

A mature API may use several layers:

```text
Internet
   ↓
Infrastructure / CDN limit
   ↓
Global API limit
   ↓
Authentication limit
   ↓
User/account limit
   ↓
Route-specific limit
   ↓
Resource-specific limit
```

Do not implement every layer unless the system actually requires it.

---

# IP-Based Rate Limiting

IP-based limits are simple and useful.

Example:

```text
IP A
 ↓
100 requests/minute
```

However, IP addresses are imperfect identifiers.

Multiple legitimate users may share one IP:

```text
School / Office / NAT
        ↓
many users
        ↓
one public IP
```

An aggressive IP limit can therefore block legitimate users.

---

# Problems with IP-Based Limits

IP addresses can be:

* shared
* dynamic
* proxied
* rotated
* spoofed at some infrastructure layers
* represented differently through proxies

Do not assume:

```text
one IP = one user
```

---

# Reverse Proxies

If the application runs behind:

* reverse proxy
* load balancer
* CDN
* ingress controller

the backend may not directly see the original client IP.

Configure trusted proxy handling correctly.

Do not blindly trust headers such as:

```text
X-Forwarded-For
```

from arbitrary clients.

Otherwise attackers may spoof their IP identity.

---

# Trusted Proxy Configuration

Only trust forwarded client information from infrastructure you actually control.

Incorrect proxy configuration can cause:

```text
attacker
 ↓
spoofed IP header
 ↓
server believes fake IP
 ↓
rate limit bypass
```

Treat proxy configuration as part of the security model.

---

# User-Based Rate Limiting

Authenticated requests can be limited by:

```text
userId
```

This is often more meaningful than IP alone.

Example:

```text
User A
 ↓
100 requests/min

User B
 ↓
100 requests/min
```

However, account-based limits should still be combined with infrastructure protection where necessary.

---

# API Key Rate Limiting

If the application supports API keys, limits can be applied per key.

Example:

```text
API key A
 ↓
10,000 requests/day

API key B
 ↓
100,000 requests/day
```

Do not expose secrets or full API keys in logs or error responses.

---

# Token-Based Rate Limiting

JWT authentication does not automatically provide rate limiting.

A token can still generate unlimited requests.

Use token-derived identity where appropriate:

```text
JWT
 ↓
userId
 ↓
rate-limit identity
```

Do not assume:

```text
has valid JWT
=
trusted unlimited traffic
```

---

# Device-Based Limiting

For security-sensitive operations, device or session signals may supplement:

* IP
* account
* token

Do not rely on device fingerprinting as a perfect identity mechanism.

It should be treated as an additional signal.

---

# Rate-Limit Algorithms

Common approaches include:

* fixed window
* sliding window
* token bucket
* leaky bucket
* concurrency limiting

Choose based on the behavior required.

---

# Fixed Window

Example:

```text
100 requests
per minute
```

At:

```text
12:00:00 → counter resets
```

A client could potentially send many requests around the boundary.

Example:

```text
12:00:59
100 requests

12:01:00
100 requests
```

This can create a burst larger than expected.

Fixed windows are simple but have boundary effects.

---

# Sliding Window

Sliding windows consider a continuously moving time period.

Example:

```text
100 requests
within the previous 60 seconds
```

This provides smoother behavior than a strict fixed window.

It may require more state or infrastructure depending on implementation.

---

# Token Bucket

A token bucket allows controlled bursts.

Conceptually:

```text
Bucket capacity = 100
Refill = 10 tokens/sec
```

Requests consume tokens.

This allows:

```text
short burst
```

while controlling sustained traffic.

Token bucket can be useful for APIs that should tolerate occasional bursts.

---

# Leaky Bucket

The leaky bucket model smooths traffic toward a controlled processing rate.

Conceptually:

```text
Incoming requests
        ↓
     Queue
        ↓
steady processing rate
```

This can be useful when downstream systems require predictable traffic.

---

# Rate Limiting vs Concurrency Limiting

These are different.

Rate limiting controls:

```text
requests per time period
```

Concurrency limiting controls:

```text
requests currently being processed
```

Example:

```text
100 requests/minute
```

does not prevent:

```text
100 expensive requests
```

from executing simultaneously.

For expensive operations, concurrency limits may be equally or more important.

---

# Combine Rate and Concurrency Limits

For expensive endpoints:

```text
Rate:
20 requests/minute

Concurrency:
2 simultaneous jobs
```

This can protect the backend more effectively than either alone.

---

# Queue + Rate Limiting

For expensive asynchronous operations:

```text
Client
 ↓
Rate Limit
 ↓
API
 ↓
Queue
 ↓
Worker concurrency limit
 ↓
External service
```

Each layer protects a different resource.

---

# Database Protection

Do not allow API traffic to translate directly into unlimited database operations.

For example:

```text
1000 requests
 ↓
1000 expensive database queries
```

can overload the database.

Rate limiting should be combined with:

* pagination
* query optimization
* indexes
* caching
* connection limits
* concurrency control

---

# Search Endpoints

Search can be expensive.

For:

```text
GET /drivers/search?q=...
```

consider:

* per-user limits
* per-IP limits
* query length limits
* pagination
* debounce on the frontend
* caching where appropriate
* database indexes

Do not let users perform unlimited expensive searches.

---

# File Uploads

File upload endpoints require more than request-count limiting.

Consider:

```text
requests/minute
+
file size
+
total upload bandwidth
+
concurrent uploads
+
allowed file types
```

A single request containing a huge file can be more expensive than hundreds of small requests.

---

# Resource-Based Limits

Not all limits should be measured in requests.

Consider limiting:

```text
MB uploaded / hour
CPU-heavy jobs / minute
reports / hour
emails / hour
SMS / hour
records processed / minute
```

This is especially important for expensive operations.

---

# Email Sending

Protect email-related endpoints against abuse.

Examples:

```text
verification email
password reset email
notification email
```

A malicious client could otherwise cause:

```text
100,000 emails
```

to be sent through the provider.

Use:

* per-account limits
* per-IP limits
* cooldowns
* daily quotas
* provider-side limits

---

# SMS Sending

SMS can be expensive.

Apply especially strict controls to:

```text
OTP
verification
password recovery
```

Consider:

```text
per phone number
per account
per IP
global application limit
```

---

# External APIs

If the backend communicates with external APIs, respect their limits.

Example:

```text
Your API
 ↓
Rate limit
 ↓
Queue
 ↓
External API
```

Do not allow a burst from your users to become an uncontrolled burst against the provider.

---

# Third-Party API Quotas

Track:

```text
requests used
remaining quota
reset time
```

where the provider exposes such information.

If the external API returns a rate-limit response, respect its retry instructions where available.

---

# HTTP 429

When a client exceeds a rate limit, return:

```text
429 Too Many Requests
```

Do not use:

```text
500 Internal Server Error
```

for normal rate-limit enforcement.

---

# Retry-After

When appropriate, communicate when the client should retry.

Example:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

The exact implementation depends on the rate-limiting strategy.

---

# Rate-Limit Response

A useful response can be:

```json
{
  "success": false,
  "message": "Too many requests. Please try again later."
}
```

Do not expose internal implementation details such as:

```text
Redis key
internal counter
server topology
```

unless intentionally required.

---

# Rate-Limit Headers

Depending on the API standard and implementation, useful headers may communicate:

```text
limit
remaining
reset
retry-after
```

Use a consistent convention throughout the API.

Do not expose information that creates unnecessary security concerns.

---

# Client Behavior

Frontend clients should react appropriately to:

```text
429
```

Possible behavior:

```text
receive 429
 ↓
read Retry-After
 ↓
wait
 ↓
retry
```

Do not immediately retry in a tight loop.

---

# Retry Storms from Frontends

A badly designed client can make rate limiting worse.

Avoid:

```text
429
 ↓
retry immediately
 ↓
429
 ↓
retry immediately
 ↓
429
...
```

Use:

* backoff
* Retry-After
* capped retries
* request deduplication
* sensible polling

---

# Rate Limiting and Polling

Aggressive polling can accidentally trigger rate limits.

Bad:

```text
GET /jobs/:id
every 100ms
```

Prefer:

```text
reasonable polling interval
```

or event-driven updates such as:

```text
SSE
WebSocket
push notification
```

when appropriate.

---

# Rate Limiting and Request Deduplication

Frontend request deduplication reduces unnecessary backend traffic.

For example:

```text
Component A
Component B
Component C
       ↓
same request
       ↓
one backend request
```

Rate limiting should not be the first solution to a frontend making duplicate requests.

Fix unnecessary traffic at the source where possible.

---

# Rate Limiting and Caching

Caching can reduce the number of requests reaching expensive resources.

For example:

```text
Client
 ↓
API
 ↓
Cache hit
 ↓
Response
```

instead of:

```text
Client
 ↓
API
 ↓
Database
```

Rate limiting and caching solve different problems and can complement each other.

---

# Distributed Rate Limiting

If the backend runs multiple instances:

```text
Load Balancer
 ├── API instance A
 ├── API instance B
 └── API instance C
```

an in-memory limiter on each instance may produce inconsistent limits.

Example:

```text
Limit = 100 requests/minute

Instance A = 100
Instance B = 100
Instance C = 100
```

The user could effectively make:

```text
300 requests/minute
```

depending on routing.

For distributed systems, use shared rate-limit state when accurate global limits are required.

---

# In-Memory Rate Limiting

In-memory rate limiting can be acceptable for:

* development
* small prototypes
* single-instance applications
* non-critical protection

But it has limitations.

Restarting the server may reset counters.

Multiple instances will have separate counters.

Do not assume an in-memory limiter provides distributed protection.

---

# Shared Rate-Limit Storage

For multiple backend instances, shared storage can synchronize limits.

Common approaches include:

* Redis
* distributed cache
* gateway/CDN rate limiting
* managed API gateway limits

Choose based on architecture.

---

# Reverse Proxy / Gateway Rate Limiting

Rate limiting can occur before the application.

Example:

```text
Internet
 ↓
CDN / Gateway
 ↓
Rate limit
 ↓
Backend
```

This can protect backend infrastructure from traffic before requests reach application servers.

Application-level limits can still provide endpoint-specific protection.

---

# Defense in Depth

A mature architecture may use:

```text
Edge protection
        ↓
Application global limit
        ↓
Authentication limit
        ↓
User/account limit
        ↓
Endpoint limit
        ↓
Concurrency limit
        ↓
Database/external service protection
```

Do not implement every layer without reason.

Use the layers appropriate to the project's risk.

---

# Rate Limit Keys

A rate-limit key determines who or what is being limited.

Examples:

```text
ip:203.0.113.10
user:12345
api-key:abc
route:/auth/login
account:12345
device:xyz
```

Composite keys can be useful.

For example:

```text
login:ip:203.0.113.10
```

and:

```text
login:user:12345
```

can enforce different limits.

---

# Key Design

Do not use sensitive raw values unnecessarily.

Avoid exposing:

```text
password
token
session secret
```

in rate-limit keys or logs.

Hash or safely represent sensitive identifiers when appropriate.

---

# Route-Specific Keys

The rate-limit identity may need to include the endpoint.

For example:

```text
user:123:/search
```

can be limited independently from:

```text
user:123:/profile
```

This prevents heavy use of one endpoint from unnecessarily consuming the user's entire API budget.

---

# Global User Limits

Sometimes the system also needs:

```text
user:123
1000 requests/minute
```

across all endpoints.

This protects against a compromised account generating traffic across many routes.

Use this alongside route-specific limits when appropriate.

---

# Authentication Bypass Risks

Do not accidentally allow unlimited requests before authentication simply because the authenticated limiter is stronger.

For example:

```text
/login
```

must have its own protection.

Attackers cannot be expected to authenticate before being rate limited.

---

# Rate Limiting Login by Username

Be careful with account-specific blocking.

If the system reveals:

```text
Account temporarily blocked
```

attackers may learn that the account exists.

Prefer generic responses where account enumeration is a concern.

---

# Progressive Delays

For sensitive operations, consider increasing delays after repeated failures.

Conceptually:

```text
Attempt 1
→ normal

Attempt 2
→ small delay

Attempt 3
→ longer delay

Attempt 4
→ longer delay
```

This can make brute-force attacks more expensive.

Do not make legitimate recovery unnecessarily frustrating.

---

# Temporary Blocking

Repeated abuse may justify temporary blocking.

For example:

```text
too many failed login attempts
 ↓
temporary restriction
 ↓
wait
 ↓
allow attempts again
```

Avoid permanent automatic blocking based solely on IP because shared networks can cause false positives.

---

# CAPTCHA / Challenge Systems

For suspicious traffic, a challenge may complement rate limiting.

Use challenges when appropriate rather than forcing every legitimate user through them.

Rate limiting should remain effective even when a challenge is not present.

---

# Bot Protection

For public applications, consider additional bot controls such as:

* behavioral analysis
* managed bot protection
* challenge mechanisms
* reputation signals
* edge filtering

Do not rely solely on a JavaScript-based frontend restriction.

Attackers can call the API directly.

---

# Client-Side Limits Are Not Security

Never rely on:

```text
button disabled
```

or:

```text
frontend waits 1 second
```

as the actual security mechanism.

Attackers can bypass the frontend completely.

Backend enforcement is authoritative.

---

# Rate Limiting Does Not Replace Validation

A request that is rate limited may still be malicious.

Always validate input separately.

Use:

```text
rate limiting
+
authentication
+
authorization
+
input validation
+
secure business logic
```

as separate controls.

---

# Rate Limiting Does Not Replace WAF / DDoS Protection

Application rate limiting can help with abuse, but large-scale denial-of-service attacks may require infrastructure-level protection.

For larger deployments, consider:

* CDN
* WAF
* load balancer
* managed DDoS protection
* API gateway

Do not expect an Express middleware to absorb a massive volumetric attack.

---

# Error Handling

Rate limiting itself should fail safely.

If the rate-limit storage system becomes unavailable, determine whether the application should:

```text
fail open
```

or:

```text
fail closed
```

based on endpoint risk.

For example:

```text
public read endpoint
```

may tolerate temporary failure differently from:

```text
login endpoint
```

Do not make this decision accidentally.

---

# Fail-Open vs Fail-Closed

## Fail Open

If the limiter cannot check:

```text
allow request
```

Advantages:

* preserves availability

Risks:

* protection disappears during limiter failure

---

## Fail Closed

If the limiter cannot check:

```text
reject request
```

Advantages:

* preserves protection

Risks:

* legitimate users may be blocked

Choose based on the protected resource and business requirements.

---

# Observability

Rate limiting should be measurable.

Track:

* number of limited requests
* endpoint
* status code
* limit policy
* client category
* response latency
* repeated offenders
* authentication failures
* distributed limiter errors

Avoid logging sensitive identifiers unnecessarily.

---

# Metrics

Useful metrics include:

```text
rate_limit_hits_total
rate_limit_rejections_total
429_responses_total
authentication_rate_limit_hits
upload_rate_limit_hits
external_api_rate_limit_hits
```

Track trends rather than only individual events.

---

# Detecting Abuse

A single 429 is not necessarily an attack.

Look for patterns:

```text
thousands of 429 responses
+
many IP addresses
+
same endpoint
+
repeated authentication failures
```

This may indicate coordinated abuse.

---

# Logging

Useful log fields include:

```text
timestamp
route
method
status
rate-limit policy
request ID
user ID when appropriate
safe client identifier
```

Do not log:

```text
password
JWT
API secret
OTP
session token
```

---

# Alerting

Consider alerts for:

* sudden increase in 429 responses
* login brute-force patterns
* unusual request spikes
* limiter storage failures
* external API quota exhaustion
* repeated abuse from a client category

Do not alert on every individual rate-limit event.

---

# Testing

Test normal traffic:

```text
request 1
request 2
request 3
```

Then test the boundary:

```text
request at limit
request above limit
```

Then test reset behavior.

---

# Distributed Testing

If the application has multiple instances, test:

```text
Instance A
Instance B
Instance C
```

and verify that shared limits behave as expected.

Do not assume local testing proves distributed correctness.

---

# Proxy Testing

If deployed behind a proxy or load balancer, verify that client identity is correctly detected.

Test:

```text
Client A
 ↓
Proxy
 ↓
Backend
```

and confirm that the backend does not accidentally treat every user as the proxy's IP.

---

# Authentication Testing

Test:

```text
many failed login attempts
```

and verify:

* rate limiting activates
* valid users are not permanently locked out
* responses do not reveal unnecessary account information
* successful authentication does not bypass global infrastructure protection

---

# Recovery Testing

Verify that limits recover correctly after:

```text
window expiration
cooldown
successful authentication
server restart
limiter restart
deployment
```

depending on the chosen algorithm and storage model.

---

# Load Testing

For important systems, simulate:

```text
normal traffic
burst traffic
sustained traffic
malicious traffic patterns
```

Measure:

* latency
* CPU
* memory
* database load
* queue depth
* 429 rate
* throughput

---

# Rate Limit Configuration

Avoid scattering hardcoded values throughout controllers.

Prefer centralized configuration.

Conceptually:

```js
const rateLimits = {
  global: {...},
  auth: {...},
  search: {...},
  upload: {...},
  reports: {...}
};
```

The exact structure should match the project.

---

# Environment Configuration

Some limits may need environment-specific values.

For example:

```text
Development
→ relaxed limits

Testing
→ deterministic limits

Production
→ carefully chosen limits
```

Do not make production security depend on developers remembering to change a hardcoded value.

---

# Avoid Magic Numbers

Avoid:

```js
100
60
10
```

without explaining what they represent.

Prefer named configuration:

```js
AUTH_LOGIN_LIMIT
AUTH_LOGIN_WINDOW
REPORT_LIMIT
REPORT_WINDOW
```

The exact naming convention should match the project.

---

# Dynamic Rate Limits

Some systems may need different limits based on:

* user role
* subscription
* API plan
* endpoint
* resource
* trust level
* internal vs external traffic

Example:

```text
VIEWER
→ normal read limits

ADMIN
→ higher operational limits

SERVICE
→ controlled machine-to-machine limits
```

Do not grant unlimited access simply because a user has a higher role.

---

# Role-Based Limits

Role-based rate limits can be useful, but authorization should still determine what the role can access.

For example:

```text
ADMIN
```

may receive a higher limit for report generation.

But:

```text
ADMIN
```

should not automatically bypass security-sensitive authentication limits.

---

# Service-to-Service Rate Limiting

Internal services can also overload each other.

Example:

```text
Service A
 ↓
Service B
```

If Service A sends unlimited requests, Service B can still fail.

Use:

* service-specific limits
* concurrency limits
* circuit breakers
* queues
* timeouts

where appropriate.

---

# Circuit Breaker Relationship

Rate limiting controls incoming volume.

Circuit breakers protect against repeatedly calling an unhealthy dependency.

Example:

```text
Rate Limit
 ↓
Service
 ↓
Circuit Breaker
 ↓
External API
```

These solve different problems and can complement each other.

---

# Timeouts

Always consider timeouts for external requests.

A rate limit does not protect against:

```text
10 requests
```

if each request remains active for:

```text
10 minutes
```

For expensive or external operations, combine:

```text
rate limiting
+
concurrency limiting
+
timeouts
```

---

# Request Body Limits

Rate limiting should be complemented by request-size limits.

Examples:

```text
JSON body size
file upload size
URL length
header size
```

Otherwise one request can consume excessive resources.

---

# Pagination

Large responses can be another resource problem.

Avoid:

```text
GET /drivers
```

returning millions of records.

Use:

* pagination
* maximum page size
* sensible defaults

Rate limiting cannot compensate for inefficient response sizes.

---

# Expensive Queries

If an endpoint accepts flexible filters or sorting, consider:

* query complexity limits
* maximum result size
* indexed fields
* pagination
* execution timeouts

Rate limiting alone does not prevent expensive individual queries.

---

# Rate Limiting and Caching

If a resource is frequently requested but changes rarely:

```text
GET /fare-matrix
```

caching may be better than simply allowing more requests.

Use:

```text
cache
+
reasonable rate limit
```

rather than:

```text
unlimited requests
```

---

# Rate Limiting and Queues

For expensive operations:

```text
Request
 ↓
Rate limit
 ↓
Queue
 ↓
Worker concurrency
```

This provides both:

```text
request frequency control
```

and:

```text
processing capacity control
```

---

# Rate Limiting and Retries

Retries must respect rate limits.

A retry system should use:

```text
backoff
+
jitter
+
Retry-After
```

where appropriate.

Never allow:

```text
429
 ↓
immediate retry
 ↓
429
 ↓
immediate retry
```

---

# Common Anti-Patterns

Avoid:

* one identical limit for every endpoint
* relying only on frontend restrictions
* relying only on IP addresses
* trusting arbitrary forwarded IP headers
* unlimited login attempts
* unlimited OTP requests
* unlimited password reset requests
* unlimited file uploads
* unlimited expensive report generation
* retrying 429 responses immediately
* using in-memory limits across many backend instances
* ignoring reverse proxies
* exposing sensitive rate-limit keys
* logging tokens or passwords
* returning 500 for rate-limit violations
* returning overly detailed security information
* permanently blocking users based solely on IP
* allowing unlimited concurrency despite request limits
* ignoring database or external-service capacity
* treating authenticated users as automatically trusted
* assuming rate limiting prevents DDoS
* hardcoding unexplained magic numbers
* creating complicated limits without measuring actual usage

---

# Production Rate-Limit Checklist

Before deploying an important API:

* [ ] Global protection exists where appropriate.
* [ ] Authentication endpoints have dedicated limits.
* [ ] Login attempts are protected.
* [ ] Password reset is protected.
* [ ] OTP/2FA operations are protected.
* [ ] Email/SMS sending is protected.
* [ ] Expensive endpoints have stronger limits.
* [ ] File uploads have size and frequency limits.
* [ ] Search endpoints are protected where necessary.
* [ ] External API quotas are respected.
* [ ] Concurrency is controlled for expensive work.
* [ ] Distributed deployments use appropriate shared state.
* [ ] Proxy configuration is correct.
* [ ] Client IP handling is trusted only from known infrastructure.
* [ ] 429 is returned correctly.
* [ ] Retry guidance is provided where appropriate.
* [ ] Frontend retries use backoff.
* [ ] Limits are configurable.
* [ ] Limits are observable.
* [ ] Sensitive information is not logged.
* [ ] Rate-limit storage failure behavior is intentional.
* [ ] Normal users are not unnecessarily blocked.
* [ ] Abuse patterns can be detected.
* [ ] Load testing has been performed where appropriate.
* [ ] Rate limiting is not being used as a replacement for authentication or authorization.

---

# Final Principle

Rate limiting should make the backend **harder to abuse without making it unpleasant to use**.

Prefer:

**different limits for different risks**

over:

**one arbitrary global number**

Prefer:

**layered protection**

over:

**one mechanism expected to solve everything**

Prefer:

**rate + concurrency + timeout controls**

for expensive operations over:

**request counting alone**

Prefer:

**shared distributed limits**

when multiple backend instances require consistent enforcement over:

**independent in-memory counters**

Prefer:

**429 + Retry-After**

over:

**silent failures**

Prefer:

**backoff and jitter**

over:

**immediate retries**

Prefer:

**account + IP + endpoint-aware protection**

over:

**IP-only protection**

Prefer:

**measured limits based on real usage**

over:

**random limits copied from another project**

The objective is not to reject requests.

The objective is to ensure that **legitimate traffic remains responsive while abusive, accidental, excessive, or unexpectedly expensive traffic cannot overwhelm the system.**

```
```
