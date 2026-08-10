# AI Skill Files

A collection of reusable AI skills designed to improve the quality, consistency, security, performance, and architecture of software development workflows.

These skills are intended to provide structured instructions and principles that an AI coding assistant can follow when working on real-world projects.

The goal is not to make AI generate more code.

The goal is to make AI generate **better software**.

---

# Overview

AI coding assistants are capable of generating functional code very quickly.

However, functional code is not always:

- well designed
- maintainable
- secure
- performant
- responsive
- accessible
- scalable
- visually polished
- appropriate for the actual project

This repository contains specialized skills that provide additional development guidelines for different areas of software engineering.

Each skill focuses on a specific responsibility rather than attempting to control every aspect of development through one massive instruction file.

---

# Skills

Skills are organized by domain.

## Frontend

### Professional Frontend Design

Path:

```text
skills/professional-frontend-design/SKILL.md

Guides AI toward professional, intentional, minimal, and visually refined frontend interfaces.

Focuses on:

visual hierarchy
typography
spacing
color systems
component design
responsive design
accessibility
animation
motion
hover states
micro-interactions
visual consistency
avoiding generic AI-generated UI patterns

The primary philosophy is:

Sharp. Smooth. Clean. Intentional. Professional.

Frontend Data Performance

Path:

skills/frontend-data-performance/SKILL.md

Provides patterns for building responsive and efficient frontend data flows.

Covers techniques such as:

request deduplication
optimistic updates
stale-while-revalidate
smart polling
retry with backoff
streaming UI
caching
request cancellation
pagination
prefetching
debouncing
throttling

These techniques should only be used when they provide a real benefit to the application.

The goal is not to make every frontend use advanced data-fetching techniques.

The goal is to make data interactions:

responsive
efficient
reliable
predictable
appropriate to the application's requirements
Backend
Secure Backend Development

Path:

skills/secure-backend-development/SKILL.md

Provides security-focused guidance for backend systems.

Covers:

authentication
authorization
password hashing
bcrypt
Argon2id
token security
JWT
refresh tokens
session management
MFA
email verification
trusted devices
reauthentication
input validation
database security
API security
rate limiting
abuse prevention
file upload security
secrets management
business logic security
audit logging
security monitoring
dependency security
OWASP awareness
common attack prevention
secure failure behavior

Security should be treated as part of the backend architecture rather than something added at the end of development.

Backend API Design

Path:

skills/backend-api-design/SKILL.md

Provides guidance for designing clean, predictable, maintainable, and consistent backend APIs.

Focuses on areas such as:

endpoint structure
resource-oriented APIs
HTTP methods
status codes
request and response formats
validation
error handling
pagination
filtering
sorting
API versioning
consistency
maintainability
API documentation
Backend Concurrency

Path:

skills/backend-concurrency/SKILL.md

Provides guidance for handling concurrent backend operations safely.

Focuses on:

race conditions
atomic operations
transactions
optimistic concurrency
pessimistic locking
unique constraints
idempotency
concurrent updates
state consistency
duplicate operations
consistency under simultaneous requests

The objective is to prevent systems from behaving incorrectly when multiple operations occur at the same time.

Backend Observability

Path:

skills/backend-observability/SKILL.md

Provides guidance for making backend systems easier to understand, monitor, debug, and operate.

Covers:

structured logging
request IDs
correlation IDs
metrics
tracing
health checks
error tracking
performance monitoring
security events
operational visibility
alerting

The objective is to make failures observable instead of forcing developers to guess what happened.

Backend Performance

Path:

skills/backend-performance/SKILL.md

Provides guidance for improving backend responsiveness and resource efficiency.

Covers areas such as:

database optimization
caching
connection pooling
pagination
query optimization
request handling
asynchronous processing
memory usage
CPU usage
response time
external service calls
request timeouts
resource limits

Performance optimizations should be based on actual bottlenecks and project requirements rather than premature optimization.

Backend Queues and Workers

Path:

skills/backend-queues-workers/SKILL.md

Provides guidance for moving expensive or asynchronous work out of the normal request-response cycle.

Useful for operations such as:

email sending
notifications
document processing
image processing
report generation
large exports
background synchronization
external API processing

Covers:

job queues
workers
retries
backoff
dead-letter handling
idempotency
duplicate jobs
job prioritization
failure handling
queue monitoring
worker concurrency

The goal is to keep APIs responsive while ensuring background work remains reliable.

Backend Rate Limiting

Path:

skills/backend-rate-limiting/SKILL.md

Provides guidance for controlling excessive requests and protecting backend resources.

Covers:

endpoint-specific limits
authentication rate limits
brute-force protection
account-based limits
IP-based limits
distributed rate limiting
burst handling
retry behavior
abuse prevention
resource protection

Rate limiting should be designed according to the risk and cost of each operation rather than applying one identical limit everywhere.

Backend Resilience

Path:

skills/backend-resilience/SKILL.md

Provides guidance for building backend systems that continue operating gracefully when dependencies or individual components fail.

Covers:

timeouts
retries
exponential backoff
circuit breakers
graceful degradation
fallback behavior
dependency failures
service recovery
queue recovery
database failures
external API failures
fault isolation

The objective is not to prevent every failure.

The objective is to ensure that failures are controlled, observable, and recoverable.

Skill Structure

Each skill is stored in its own directory:

skills/
├── backend-api-design/
│   └── SKILL.md
│
├── backend-concurrency/
│   └── SKILL.md
│
├── backend-observability/
│   └── SKILL.md
│
├── backend-performance/
│   └── SKILL.md
│
├── backend-queues-workers/
│   └── SKILL.md
│
├── backend-rate-limiting/
│   └── SKILL.md
│
├── backend-resilience/
│   └── SKILL.md
│
├── frontend-data-performance/
│   └── SKILL.md
│
├── professional-frontend-design/
│   └── SKILL.md
│
└── secure-backend-development/
    └── SKILL.md

Each skill is intentionally separated so it can be used independently.

Why Separate Skills?

A single massive skill file would eventually become difficult to maintain.

Different projects require different concerns.

For example:

Landing page
→ Professional Frontend Design

CRUD application
→ Backend API Design
→ Secure Backend Development

Real-time dashboard
→ Backend Performance
→ Backend Concurrency
→ Frontend Data Performance

Large asynchronous system
→ Backend Queues and Workers
→ Backend Resilience
→ Backend Observability

Skills should be selected according to the project's requirements.

Design Philosophy

These skills follow several general principles.

1. Do not over-engineer

Advanced techniques should not be implemented simply because they exist.

A small application does not automatically need:

distributed caching
queues
workers
streaming
complex authorization systems
circuit breakers
distributed locks
sophisticated observability infrastructure

Use complexity when the project actually benefits from it.

2. Prefer maintainability

Code should be understandable by the developers who will maintain it.

Avoid unnecessary abstractions.

Avoid creating files, services, utilities, or architectural layers without a meaningful reason.

A smaller system should generally have a smaller architecture.

3. Security is not optional

Security should be considered during implementation rather than after functionality is complete.

Important security mechanisms should become normal development practices.

4. Performance should serve the user

Performance improvements should improve actual user experience or system reliability.

Avoid optimizing arbitrary numbers simply because they can be measured.

Prioritize:

responsiveness
predictable behavior
efficient resource usage
reliability
scalability where actually required
5. Design should serve the product

Frontend interfaces should not look like generic AI-generated interfaces.

Avoid adding visual effects simply because they are popular.

Design decisions should support:

usability
hierarchy
identity
clarity
interaction
accessibility
Using the Skills

When using these skills with an AI coding assistant:

Identify which area of the project is being worked on.
Load the relevant skill.
Apply only the principles that are relevant to the current task.
Inspect the existing project before making architectural changes.
Preserve existing intentional decisions.
Avoid unnecessary complexity.
Prefer maintainable implementations.
Test the resulting behavior.

Multiple skills can be used together when their concerns overlap.

For example:

Secure Backend Development
        +
Backend API Design
        +
Backend Concurrency
        +
Backend Performance

may be appropriate for a larger backend system.

Important Principle

These skills are guidelines, not rigid rules.

They should help an AI make better engineering decisions.

They should not force an architecture onto a project when that architecture is unnecessary.

When two approaches are both valid, prefer the approach that is:

simpler
safer
clearer
easier to maintain
appropriate for the project's scale
easier to test
easier to operate
Project Philosophy

The purpose of this repository is simple:

Make AI-assisted software development produce software that feels intentionally engineered rather than merely generated.

That means:

Frontend should be designed.

Backend should be secure.

APIs should be predictable.

Systems should be resilient.

Performance should be intentional.

Complexity should have a reason.

Security should be normal.

And the final product should feel like a professional built it.