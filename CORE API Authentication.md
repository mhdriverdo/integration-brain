---
title: CORE API - Authentication
type: api-component
domain: core-api
tags:
  - core-api
  - authentication
  - api
  - bearer-token
  - organization
  - security
related:
  - "[[CORE API]]"
  - "[[Organization]]"
  - "[[Integration]]"
---

# CORE API - Authentication

## Overview

The **CORE API** requires authentication before an integrator can access protected DRAIVER endpoints.

The API uses **Bearer tokens** in the `Authorization` header.

There are two authentication patterns documented:

1. **Short-lived authentication tokens** obtained through `/authenticate`.
2. **Long-lived tokens** provided by the platform.

The authentication model is important for integrations because every request made to CORE API must operate within an authenticated session and, when applicable, a specific [[Organization]].

---

# Short-Lived Token Authentication

The standard authentication flow starts by providing a username and password to the authentication endpoint.

```text
Username + Password
        ↓
POST /authenticate
        ↓
Authentication response
        ↓
Bearer token
        ↓
CORE API requests
```

The issued token is short-lived and should not be assumed to remain valid for more than approximately one hour.

Therefore, an integrator that stores and reuses the token must implement a mechanism to obtain a new authorization when the token expires.

---

## Authentication Endpoint

```text
POST /authenticate
```

Example request:

```bash
curl --location 'http://core.prod.appservice.draiver.net/authenticate'   --header 'Content-Type: application/json'   --data '{
    "username": "your_user",
    "password": "your_password"
  }'
```

### Important documentation note

The provided documentation describes the endpoint as:

```text
POST /authenticate
```

but the example shows:

```text
curl -X GET ".../authenticate"
```

This is an inconsistency in the source documentation. For an actual integration, the HTTP method should be verified against the deployed CORE API/environment rather than relying solely on the example.

---

# Authentication Response

A successful authentication response contains information about the authenticated user, session, token, organization and permissions.

Representative structure:

```json
{
  "user": {
    "id": "EewFkSlVm_eY4QK0AD5BeQ",
    "username": "user",
    "locale": "en_US"
  },
  "sessionId": "fca5d83f-a654-4352-b06a-66459251af96",
  "token": "your_authentication_token",
  "refreshToken": "your_authentication_token",
  "expirationDate": "2024-03-25T18:57:59Z",
  "currentOrganization": {
    "organizationId": "your_organization",
    "organizationName": "Mascot Autos",
    "permissions": [
      "ACTION_PERSON_MOVE_CREATE",
      "ACTION_PERSON_MOVE_DELETE",
      "WORKORDER_VIEW"
    ],
    "featureFlags": [
      "TEST_ALWAYS_ON"
    ]
  }
}
```

---

# Important Response Fields

| Field | Purpose |
|---|---|
| `user` | Information about the authenticated user |
| `sessionId` | Identifier for the authenticated session |
| `token` | Bearer token used for API authentication |
| `refreshToken` | Token associated with refreshing authentication |
| `expirationDate` | Token expiration timestamp |
| `currentOrganization` | Organization/tenant selected for the session |
| `permissions` | Permissions available within the organization |
| `featureFlags` | Feature flags enabled for the organization |

---

# Organization / Tenant

The `organizationId` contained inside `currentOrganization` represents the tenant context of the authenticated session.

```text
Authentication
      ↓
currentOrganization
      ↓
organizationId
      ↓
Tenant context
```

This is an important concept for integrations because the same user can potentially have access to multiple organizations.

The `organizationId` should therefore be treated as a fundamental piece of context when making CORE API requests.

Related: [[Organization]]

---

# Making Authenticated Requests

Once a token has been obtained, include it in the `Authorization` header.

```bash
curl --location 'http://core.prod.appservice.draiver.net/action'   --header 'Authorization: Bearer <apiToken>'   --header 'Content-Type: application/json'
```

The standard pattern is:

```http
Authorization: Bearer <apiToken>
```

Conceptually:

```text
Authenticate
     ↓
Receive token
     ↓
Authorization: Bearer <token>
     ↓
CORE API
```

---

# Multiple Organizations

When a user has access to multiple organizations, the integration may need to authenticate within a specific organization.

The documented endpoint is:

```text
/orgs/{organizationId}
```

Example:

```bash
curl --location 'http://core.prod.appservice.draiver.net/orgs/your_org_id'   --header 'Content-Type: application/json'   --data '{
    "username": "your_user",
    "password": "your_password"
  }'
```

The organization-specific authentication provides a session context containing:

- Organization information.
- Organization-specific permissions.
- Feature flags.
- Authentication/session information.

Conceptually:

```text
User
 │
 ├── Organization A
 │
 ├── Organization B
 │
 └── Organization C
        ↓
Authenticate within selected organization
        ↓
Organization-specific session
```

---

# Long-Lived Token Authentication

CORE API documentation also describes a **long-lived token** authentication mechanism.

In this model, the platform provides a token that can be reused across multiple API requests.

The token is supplied in the same Bearer format:

```http
Authorization: Bearer <apiToken>
```

Example:

```bash
curl --location 'http://core.prod.appservice.draiver.net/action'   --header 'Authorization: Bearer <apiToken>'   --header 'Content-Type: application/json'
```

Unlike the short-lived authentication flow, the long-lived token is intended to remain valid for an extended period.

---

# Short-Lived vs Long-Lived Tokens

| Characteristic | Short-lived token | Long-lived token |
|---|---|---|
| Obtained through | Authentication flow | Provided by platform |
| Lifetime | Approximately ≤ 1 hour | Extended |
| Reuse | Requires re-authentication/refresh handling | Can be reused |
| Typical concern | Token expiration | Credential security |
| Authorization header | `Bearer <token>` | `Bearer <token>` |

The exact expiration and lifecycle behavior should be verified against the environment being used by the integration.

---

# Authentication in an Integration

A typical integration should think about authentication as a separate concern from business API calls.

```text
Integration
    │
    ├── Authentication
    │      ↓
    │   Token
    │
    └── CORE API requests
           │
           ├── Create Move
           ├── Update Move
           ├── Query WorkOrder
           └── ...
```

The token should generally be obtained before making protected CORE API requests.

For short-lived tokens:

```text
Request
   ↓
Token valid?
   │
   ├── Yes → API request
   │
   └── No
        ↓
   Authenticate again
        ↓
   New token
        ↓
   API request
```

---

# Permissions

The authentication response includes organization-specific permissions.

Example:

```json
"permissions": [
  "ACTION_PERSON_MOVE_CREATE",
  "ACTION_PERSON_MOVE_DELETE",
  "WORKORDER_VIEW"
]
```

Authentication and authorization are therefore related but distinct concepts:

```text
Authentication
    ↓
"Who are you?"
    ↓
Token / session

Authorization
    ↓
"What can you do?"
    ↓
Permissions
```

An authenticated request can still fail if the authenticated organization/user does not have the required permission.

---

# Feature Flags

The organization context can also contain feature flags.

Example:

```json
"featureFlags": [
  "TEST_ALWAYS_ON"
]
```

Feature flags can affect which functionality is available to an integration or organization.

When troubleshooting behavior that differs between organizations or environments, feature flags may therefore be relevant.

---

# Common Authentication Errors

| Status | Meaning | Typical investigation |
|---|---|---|
| `400` | Bad Request | Validate request structure/body |
| `401` | Unauthorized | Check credentials, token and expiration |
| `403` | Forbidden | Check permissions and organization |
| `429` | Too Many Requests | Reduce request frequency / investigate rate limits |

### 401 Unauthorized

Common things to check:

```text
Token present?
     ↓
Correct Authorization header?
     ↓
Correct Bearer format?
     ↓
Token expired?
     ↓
Token generated for correct context?
```

### 403 Forbidden

The token may be valid, but the user/session may not have permission to access the requested resource.

Check:

- Organization.
- User permissions.
- Endpoint permissions.
- Feature flags where relevant.

### 429 Too Many Requests

The API is rejecting requests because the request rate is too high.

An integration should handle this explicitly rather than treating it as a permanent API failure.

---

# Security Best Practices

API credentials and tokens should be treated as secrets.

Recommended practices:

- Rotate credentials regularly.
- Store credentials in environment variables or a secrets manager.
- Never commit credentials to source control.
- Never expose tokens in public repositories.
- Avoid logging authentication tokens.
- Avoid including credentials in error messages.
- Implement proper authentication failure handling.

---

# Integration Troubleshooting Checklist

When a CORE API request fails due to authentication:

```text
1. Is the endpoint correct?
        ↓
2. Is the HTTP method correct?
        ↓
3. Is the request body valid?
        ↓
4. Are the username/password valid?
        ↓
5. Was a token successfully generated?
        ↓
6. Is the token expired?
        ↓
7. Is Authorization: Bearer <token> present?
        ↓
8. Is the token associated with the correct Organization?
        ↓
9. Does the organization/user have the required permission?
        ↓
10. Is a feature flag affecting the behavior?
```

---

# Integration Knowledge

For an integration using CORE API, authentication is usually one of the first things to establish before implementing business flows.

A useful mental model is:

```text
Credentials
    ↓
Authentication
    ↓
Token
    ↓
Organization context
    ↓
Permissions
    ↓
CORE API
    ↓
Business operation
```

This authentication layer is independent of whether the integration subsequently works with [[Procurement]], [[Inbound]], [[Webhook]], or other DRAIVER services.

---

# Documentation Gaps / Things to Verify

The current documentation leaves several implementation details that should be verified before treating them as contractual behavior:

- Whether `/authenticate` is `POST` or `GET` in each environment.
- Exact authentication request schema.
- Exact token expiration behavior.
- Whether `refreshToken` is currently supported and how it is used.
- Exact behavior of organization-specific authentication.
- How long-lived tokens are generated and revoked.
- Whether long-lived tokens are scoped to an organization.
- Rate limits and retry guidance.
- Exact permission requirements for individual CORE API endpoints.

---

# Related Notes

- [[CORE API]]
- [[Organization]]
- [[Integration]]
- [[Inbound]]
- [[Procurement]]
- [[Webhook]]
- [[Move]]
- [[WorkOrder]]

# Tags

#core-api #authentication #api #bearer-token #organization #security
