# Threat-Based Security and Destructive Action Authorization

## Purpose

This skill defines a flexible framework for deciding **how much security an application should require before allowing an operation**.

The framework is based on:

* Threat level
* Data sensitivity
* Destructive impact
* Recoverability
* Scope of impact
* Authorization level
* Authentication strength
* Device trust
* Recent/step-up authentication
* User friction

The goal is **not** to require maximum security everywhere.

Instead:

> Use the minimum security controls that are appropriate for the risk of the operation.

A simple portfolio should not require a trusted device and 2FA just to edit a paragraph.

A banking system absolutely should not allow an administrator to delete an account using only an existing session.

---

# 1. Core Security Model

Separate these concepts.

## Authentication

Answers:

> "Who are you?"

Examples:

* Password
* Passkey
* OAuth
* Session
* JWT
* API credential

---

## Authorization

Answers:

> "Are you allowed to perform this operation?"

Examples:

* User
* Admin
* Moderator
* Superadmin
* Owner

Authentication does not automatically grant authorization.

---

## Device Trust

Answers:

> "Has this particular device previously passed our device-verification policy?"

A trusted device may reduce authentication friction.

It does **not** automatically mean the user is authorized to perform dangerous operations.

Device trust is optional.

Projects that do not need device recognition should not implement it.

---

## Step-Up Authentication

Answers:

> "Has this authenticated user recently proven their identity strongly enough for this sensitive operation?"

Examples:

* Re-enter password
* Enter 2FA/TOTP code
* Use passkey
* WebAuthn verification
* Biometric verification
* Security key

Step-up authentication is optional and should be introduced when the operation justifies the additional security.

---

## Audit Logging

Answers:

> "What happened, who performed it, when, and against what?"

Sensitive operations should generally produce an audit event.

Examples:

```text
ADMIN_PASSWORD_CHANGED
ADMIN_DELETED
DEVICE_REVOKED
DEVICE_LOGGED_OUT
PERMISSION_CHANGED
SECURITY_SETTING_CHANGED
```

---

# 2. Risk Factors

Determine the risk of an operation using several dimensions.

## A. Data Sensitivity

How sensitive is the affected information?

### Low

Examples:

* Public portfolio text
* Public images
* Public descriptions

### Medium

Examples:

* Internal drafts
* Private profile information
* Non-public analytics

### High

Examples:

* Passwords
* Authentication secrets
* 2FA configuration
* Recovery codes
* Personal information
* Payment information
* Access tokens

### Critical

Examples:

* Master credentials
* Security keys
* Encryption keys
* Organization ownership
* Root/superadmin privileges

---

# 3. Destructive Impact

Determine what happens if the operation is abused.

## None

The operation can easily be reversed.

Examples:

* Change display text
* Change theme
* Update a description

---

## Low

The operation changes data but has minimal consequences.

Examples:

* Edit a normal profile
* Change a portfolio section

---

## Medium

The operation could cause meaningful disruption.

Examples:

* Disable an account
* Remove an important configuration
* Change permissions

---

## High

The operation can significantly affect security, availability, or other users.

Examples:

* Logout another user's devices
* Revoke access
* Change another administrator's password
* Remove permissions

---

## Critical

The operation can cause severe or difficult-to-recover consequences.

Examples:

* Delete an administrator
* Delete organization ownership
* Reset a security administrator
* Delete critical infrastructure
* Rotate a master secret
* Permanently destroy important data

---

# 4. Recoverability

Consider whether the action can be undone.

### Easily reversible

Examples:

```text
Edit → restore previous value
```

Minimal additional protection may be appropriate.

### Recoverable

Requires an additional action or administrator intervention.

Use stronger authorization.

### Difficult to recover

Requires backups, support intervention, or manual restoration.

Use strong step-up authentication.

### Irreversible

Cannot reliably be undone.

Use the strongest available controls.

---

# 5. Scope of Impact

Ask:

> "How many things or people can this operation affect?"

### Self

Only affects the current user.

Example:

```text
Change my display name
```

### Single resource

Affects one object.

Example:

```text
Delete one portfolio image
```

### Another user

Affects another account.

Example:

```text
Disable another admin
```

### Multiple users

Affects multiple accounts.

Example:

```text
Logout all users
```

### System-wide

Affects the entire application.

Example:

```text
Change global authentication policy
```

The wider the scope, the stronger the authorization requirements should generally become.

---

# 6. Threat Levels

Use the following baseline threat levels.

## LEVEL 0 — Public / No Security

Use for operations that do not require authentication.

Examples:

* View public portfolio
* Read public content
* Public contact form

Security:

```text
Authentication: NONE
Authorization: NONE
Step-up: NONE
Audit: OPTIONAL
```

---

# LEVEL 1 — Normal Authenticated Operation

Use for ordinary actions performed by an authenticated user.

Examples:

* Edit own profile
* Update ordinary content
* Upload personal content
* Change preferences

Security:

```text
Authentication: REQUIRED
Authorization: REQUIRED
Trusted device: OPTIONAL
Step-up: NONE
Audit: OPTIONAL
```

---

# LEVEL 2 — Sensitive Operation

Use when an operation exposes sensitive information or changes meaningful account state.

Examples:

* View security settings
* View private information
* Change important account settings
* Manage ordinary resources

Security:

```text
Authentication: REQUIRED
Authorization: REQUIRED
Trusted device: OPTIONAL
Step-up: OPTIONAL
Audit: RECOMMENDED
```

Whether step-up authentication is required depends on the application's threat model.

---

# LEVEL 3 — High-Risk Operation

Use when the operation can significantly affect security, access, permissions, or another user.

Examples:

* Change another user's permissions
* Disable another user
* Revoke another device
* Logout another device
* Change another administrator's password

Recommended:

```text
Authentication: REQUIRED
Authorization: REQUIRED
Trusted device: OPTIONAL
Step-up: RECOMMENDED
Audit: REQUIRED
Confirmation: RECOMMENDED
```

A project may require:

```text
Trusted device + recent 2FA
```

if its threat model warrants it.

A lower-risk project may instead use:

```text
Authenticated session + password confirmation
```

---

# LEVEL 4 — Critical / Destructive Operation

Use for operations that are highly destructive, difficult to recover, or capable of significantly compromising the system.

Examples:

* Delete administrator
* Delete organization
* Change ownership
* Reset another administrator's security credentials
* Remove the last privileged administrator
* Delete critical data
* Change root security configuration

Recommended:

```text
Authentication: REQUIRED
Authorization: REQUIRED
Trusted device: OPTIONAL
Fresh/step-up authentication: STRONGLY RECOMMENDED
2FA/passkey/security key: RECOMMENDED
Explicit confirmation: REQUIRED
Audit log: REQUIRED
```

For particularly sensitive systems:

```text
Authenticated session
        ↓
Trusted device
        ↓
Fresh 2FA/passkey
        ↓
Explicit confirmation
        ↓
Authorization check
        ↓
Execute
        ↓
Audit log
```

---

# 7. Security Controls Are Composable

Do not assume every application needs every control.

Controls should be selected independently.

Possible controls:

```text
Authentication
Authorization
Password re-entry
2FA
Passkey
Trusted device
Recent-authentication requirement
Explicit confirmation
Typed confirmation
Approval workflow
Dual authorization
Audit logging
Rate limiting
Session invalidation
```

For example:

### Simple portfolio

```text
Edit portfolio
→ authenticated admin
```

No trusted devices required.

---

### Small SaaS

```text
Delete project
→ authenticated user
→ project authorization
→ confirmation
```

---

### Admin panel

```text
Delete user
→ authenticated admin
→ authorization
→ recent password/2FA verification
→ confirmation
→ audit log
```

---

### High-security administration system

```text
Delete privileged administrator
→ authenticated session
→ privileged authorization
→ trusted device
→ fresh 2FA/passkey
→ explicit confirmation
→ audit log
→ optional second administrator approval
```

---

# 8. Trusted Devices Are Optional

Do not automatically introduce trusted devices.

Use them when the application benefits from:

* Device recognition
* Reduced 2FA friction
* Administrator security
* Long-lived privileged sessions
* Security-sensitive account management
* Known-device policies

A project may instead use:

```text
Password + 2FA every login
```

and have no concept of trusted devices.

Another project may use:

```text
Password
+
recognized trusted device
```

and only require 2FA for unfamiliar devices.

Another may use:

```text
Password
+
trusted device
+
step-up 2FA for destructive operations
```

All three are valid depending on the threat model.

---

# 9. Trusted Device ≠ Authorization

Never use:

```js
isTrusted === true
```

as a substitute for authorization.

Correct:

```text
Authenticated?
        ↓
Authorized?
        ↓
Trusted device if policy requires it
        ↓
Step-up authentication if policy requires it
        ↓
Execute
```

A trusted device does not grant permissions.

---

# 10. Step-Up Authentication

Step-up authentication should be scoped to the sensitive operation.

Example:

```text
Normal session
    ↓
User clicks "Delete Admin"
    ↓
Security policy requires step-up
    ↓
2FA verification
    ↓
Short-lived elevated authorization
    ↓
Delete Admin
```

The elevated state should expire.

Example:

```text
sensitiveAuthUntil = now + 5 minutes
```

The exact duration should be configurable.

For extremely sensitive actions, require fresh authentication every time.

---

# 11. Avoid Requiring 2FA for Every Dangerous Button

Do not blindly implement:

```text
Delete device → 2FA
Delete another device → 2FA
Logout device → 2FA
Untrust device → 2FA
Delete user → 2FA
Change password → 2FA
```

without considering context.

Instead define a policy.

Example:

```js
securityPolicy = {
    deleteDevice: {
        level: 3,
        stepUp: true
    },

    logoutDevice: {
        level: 3,
        stepUp: false
    },

    deleteAdmin: {
        level: 4,
        stepUp: true,
        confirmation: true,
        audit: true
    }
};
```

The application decides which controls are appropriate.

---

# 12. Security Policy Should Be Configurable

A security system should not hardcode one universal policy.

Example:

```js
const securityPolicy = {
    operation: "DELETE_ADMIN",

    threatLevel: 4,

    authentication: true,

    authorization: ["SUPERADMIN"],

    trustedDevice: true,

    stepUpAuthentication: "2FA",

    recentAuthenticationWindow: 300,

    confirmation: true,

    auditLog: true
};
```

Another application could define:

```js
const securityPolicy = {
    operation: "DELETE_POST",

    threatLevel: 1,

    authentication: true,

    authorization: ["ADMIN"],

    trustedDevice: false,

    stepUpAuthentication: false,

    confirmation: false,

    auditLog: false
};
```

The framework adapts to the application.

---

# 13. Authorization Must Be Enforced Server-Side

Never rely on frontend controls.

This is insufficient:

```js
if (isSuperAdmin) {
    showDeleteButton();
}
```

The backend must independently verify:

```text
Authenticated session
        ↓
Valid session
        ↓
Correct user
        ↓
Correct role/permission
        ↓
Security policy
        ↓
Trusted-device requirement
        ↓
Step-up requirement
        ↓
Operation
```

A hidden button is UX.

A backend authorization check is security.

---

# 14. Target Ownership Must Be Verified

For operations affecting another resource, verify that the target belongs to the correct security domain.

Example:

```text
Authenticated Superadmin
        ↓
Target device
        ↓
Does this device belong to the target admin?
        ↓
YES → continue
NO → reject
```

Never blindly trust:

```http
x-device-id: some-device-id
```

or:

```text
/admin/:id
```

from the client.

The server must establish the relationship.

---

# 15. Destructive Operations Should Be Audited

At minimum record:

```text
actor
action
target
timestamp
result
```

For security-sensitive systems also consider:

```text
actor admin ID
target admin ID
target resource ID
device ID
session ID
IP address
user agent
authentication method
security level
success/failure
reason
```

Never log:

* Passwords
* 2FA secrets
* Recovery codes
* Authentication tokens
* Private cryptographic material

---

# 16. Failed Sensitive Operations Should Also Be Considered

A failed attempt to perform a critical operation can itself be security-relevant.

Examples:

```text
DELETE_ADMIN_FAILED
CHANGE_ADMIN_PASSWORD_FAILED
DEVICE_REVOKE_FAILED
STEP_UP_AUTH_FAILED
```

This allows security monitoring to detect suspicious behavior.

---

# 17. Example Security Matrix

| Operation                          | Threat | Destructive | Auth | Authorization | Trusted Device | Step-Up              | Confirmation | Audit       |
| ---------------------------------- | -----: | ----------: | ---- | ------------- | -------------- | -------------------- | ------------ | ----------- |
| View public content                |      0 |        None | No   | No            | No             | No                   | No           | No          |
| Edit own profile                   |      1 |         Low | Yes  | Yes           | No             | No                   | No           | Optional    |
| Edit portfolio                     |      1 |         Low | Yes  | Yes           | No             | No                   | No           | Optional    |
| View security settings             |      2 |      Medium | Yes  | Yes           | Optional       | Optional             | No           | Recommended |
| Change own password                |    2–3 |      Medium | Yes  | Yes           | Optional       | Recommended          | No           | Yes         |
| Logout another device              |      3 |        High | Yes  | Yes           | Optional       | Policy-dependent     | Optional     | Yes         |
| Revoke another device              |      3 |        High | Yes  | Yes           | Optional       | Recommended          | Optional     | Yes         |
| Change another admin's password    |    3–4 |        High | Yes  | Yes           | Optional       | Strongly recommended | Yes          | Yes         |
| Change admin permissions           |    3–4 |        High | Yes  | Yes           | Optional       | Strongly recommended | Yes          | Yes         |
| Delete administrator               |      4 |    Critical | Yes  | Yes           | Optional       | Strongly recommended | Required     | Required    |
| Delete organization                |      4 |    Critical | Yes  | Yes           | Optional       | Strongly recommended | Required     | Required    |
| Change root security configuration |      4 |    Critical | Yes  | Yes           | Optional       | Strongly recommended | Required     | Required    |

This is a **baseline**, not a universal law.

---

# 18. Recovery Must Be Considered

Security controls should not create an impossible recovery situation.

Before implementing a critical requirement, ask:

> "What happens if the legitimate administrator loses the required authenticator/device?"

Possible recovery mechanisms:

* Backup codes
* Recovery keys
* Secondary trusted device
* Another authorized administrator
* Recovery workflow
* Manual support verification
* Emergency administrative procedure

Recovery mechanisms must themselves be protected.

A security system that cannot distinguish:

```text
legitimate owner locked out
```

from:

```text
attacker trying to bypass security
```

needs a carefully designed recovery process.

---

# 19. Principle of Least Privilege

Do not give an administrator more authority than necessary.

Example:

```text
ContentAdmin
    → edit portfolio

UserAdmin
    → manage users

SecurityAdmin
    → manage authentication/security

Superadmin
    → system-wide administration
```

Sensitive operations should require the smallest appropriate privilege set.

---

# 20. Principle of Least Friction

Security should match risk.

Do not ask:

> "How can we make this as secure as possible?"

Ask:

> "What security controls are justified by the consequences of this operation?"

A low-risk operation should remain convenient.

A critical operation should intentionally be inconvenient.

That inconvenience is a security feature.

---

# 21. Recommended Decision Process

When implementing a new operation:

### Step 1

Identify the resource.

```text
What is being changed?
```

### Step 2

Identify the actor.

```text
Who is performing the operation?
```

### Step 3

Determine scope.

```text
Self
Single resource
Another user
Multiple users
System-wide
```

### Step 4

Determine destructiveness.

```text
Reversible
Recoverable
Difficult to recover
Irreversible
```

### Step 5

Determine threat level.

```text
0
1
2
3
4
```

### Step 6

Select authentication requirements.

```text
Normal authentication
Password re-entry
2FA
Passkey
Trusted device
```

### Step 7

Select authorization requirements.

```text
Role
Permission
Ownership
Resource relationship
```

### Step 8

Add confirmation if appropriate.

```text
Normal confirmation
Explicit confirmation
Typed confirmation
Multi-person approval
```

### Step 9

Add audit logging.

Especially for:

```text
Level 3
Level 4
```

### Step 10

Design recovery.

Ask:

```text
What happens if the legitimate user loses access?
```

---

# 22. Golden Rule

Never make security requirements universal when the threat model is not universal.

Use:

```text
Risk
  ↓
Threat level
  ↓
Security policy
  ↓
Authentication
  ↓
Authorization
  ↓
Optional trusted device
  ↓
Optional step-up authentication
  ↓
Confirmation
  ↓
Audit
  ↓
Operation
```

The application's security policy determines which layers are actually required.

The system should be **secure by design without being unnecessarily paranoid by default**.

Unless, of course, the developer specifically requests paranoid mode.

Then:

```text
¯\_(ツ)_/¯
```

Enable all the locks.
