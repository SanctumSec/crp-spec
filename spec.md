# Credential Resolution Protocol (CRP) v0.1

**Status:** Draft  
**Date:** 2026-02-13  
**Authors:** SanctumAI Contributors  
**Target:** Model Context Protocol (MCP) Working Group  

---

## 1. Abstract

The Credential Resolution Protocol (CRP) defines a standard interface for resolving credentials needed by MCP servers to authenticate with downstream APIs and services. CRP operates within MCP's `experimental` capability extension mechanism, enabling any conforming vault provider—local or remote—to supply credentials on demand with policy enforcement, audit logging, and automatic lease management.

CRP addresses the "last mile" credential problem: MCP OAuth 2.1 authenticates clients to servers, but provides no standard for how servers obtain credentials for the downstream services they proxy. CRP fills this gap.

---

## 2. Introduction & Motivation

### 2.1 The Problem

Modern AI agents interact with dozens of APIs through MCP servers. Each server needs credentials for downstream services—API keys, OAuth tokens, database connection strings. Today, these credentials are managed ad hoc:

- Hard-coded in environment variables
- Copy-pasted into configuration files  
- Shared across agents with no access control
- Never rotated, never audited, never revoked

This is the status quo documented in [GitHub Issue #754](https://github.com/modelcontextprotocol/servers/issues/754) on the MCP servers repository, which requests credential management best practices. [Working Group Discussion #220](https://github.com/modelcontextprotocol/specification/discussions/220) further demonstrates market demand: Glama manages credentials across ~3,000 MCP server instances, and community members have independently built `CredentialResolver` interfaces for just-in-time credential fetching.

### 2.2 The Gap

MCP's authentication model covers one hop:

```
Client ──(OAuth 2.1)──▶ MCP Server ──( ??? )──▶ Downstream API
```

CRP standardizes the `???`:

```
Client ──(OAuth 2.1)──▶ MCP Server ──(CRP)──▶ Credential Provider ──▶ Downstream API
```

### 2.3 Design Goals

1. **Simple.** A basic CRP provider is buildable in a weekend.
2. **Open.** No vendor lock-in. Any conforming provider works.
3. **Secure.** Short-lived leases, least privilege, mandatory audit trails.
4. **Incremental.** Opt-in via MCP's `experimental` capability—zero breaking changes.
5. **Universal.** Supports local vaults, remote hosted providers, and hybrid deployments.

---

## 3. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

| Term | Definition |
|------|-----------|
| **CRP Provider** | A service or library that resolves credential requests. May be local (in-process vault) or remote (network service). |
| **CRP Consumer** | An MCP server that requests credentials from a CRP Provider to authenticate with downstream services. |
| **Credential** | An authentication artifact: API key, OAuth bearer token, client certificate, connection string, or custom type. |
| **Lease** | A time-bounded grant of a credential. Has a TTL, can be renewed, and expires automatically. |
| **Lease ID** | An opaque string uniquely identifying an active lease. |
| **Service Identifier** | A namespaced string identifying the downstream service (e.g., `github.com/api`, `aws/s3`, `postgres/prod`). |
| **Scope** | A set of permissions or access boundaries associated with a credential (e.g., `repo:read`, `s3:GetObject`). |
| **Policy** | Rules governing which consumers may access which credentials under what conditions. |
| **Resolution** | The act of a CRP Provider fulfilling a credential request—fetching, generating, or decrypting the credential material. |

---

## 4. Protocol Overview

### 4.1 Architecture

```
┌─────────────┐         ┌─────────────┐         ┌──────────────────┐
│             │  MCP     │             │  CRP    │                  │
│  MCP Client │────────▶│  MCP Server │────────▶│  CRP Provider    │
│  (Agent)    │  OAuth   │  (Consumer) │ resolve │  (Vault)         │
│             │  2.1     │             │         │                  │
└─────────────┘         └──────┬──────┘         └────────┬─────────┘
                               │                         │
                               │  Authenticated          │  Credential
                               │  API calls              │  Storage
                               ▼                         ▼
                        ┌─────────────┐         ┌──────────────────┐
                        │ Downstream  │         │  Secrets Backend │
                        │ API/Service │         │  (Keychain, HSM, │
                        │             │         │   KMS, File, etc)│
                        └─────────────┘         └──────────────────┘
```

### 4.2 Flow Summary

1. During MCP `initialize`, client and server exchange `experimental.crp` capabilities.
2. MCP server discovers its CRP Provider (local library, Unix socket, or HTTP endpoint).
3. When the server needs a downstream credential, it sends `crp/resolve` to the provider.
4. The provider evaluates policy, resolves the credential, and returns it with a lease.
5. The server uses the credential, respects the lease TTL, and renews or releases as needed.
6. All operations are audit-logged by the provider.

### 4.3 Transport

CRP messages are JSON-RPC 2.0, consistent with MCP's own transport. Providers MUST support at least one of:

- **In-process** — Direct function calls (for library-based providers)
- **Stdio** — JSON-RPC over stdin/stdout (for subprocess providers)
- **HTTP** — JSON-RPC over HTTP/HTTPS (for remote providers)

The transport selection is a deployment concern, not a protocol concern. All message schemas are transport-agnostic.

---

## 5. Capability Negotiation

### 5.1 MCP Integration Seam

CRP uses MCP's `experimental` capability field, requiring zero changes to the MCP specification.

During the MCP `initialize` handshake, a CRP-aware server advertises support:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "tools": {},
      "experimental": {
        "crp": {
          "version": "0.1",
          "provider": "sanctum-local",
          "features": ["resolve", "lease", "revoke", "list"]
        }
      }
    },
    "serverInfo": {
      "name": "example-server",
      "version": "1.0.0"
    }
  }
}
```

A CRP-aware client MAY also advertise CRP capabilities to indicate it can participate in credential workflows (e.g., prompting the user for consent):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "experimental": {
        "crp": {
          "version": "0.1",
          "features": ["consent"]
        }
      }
    },
    "clientInfo": {
      "name": "example-client",
      "version": "1.0.0"
    }
  }
}
```

### 5.2 Version Negotiation

The `version` field uses semantic versioning (`MAJOR.MINOR`). Providers and consumers MUST agree on the same major version. Minor version differences are backward-compatible.

When versions differ:

- If major versions match, use the lower minor version's feature set.
- If major versions differ, the provider MUST reject the connection with error code `-32001` (version mismatch).

### 5.3 Feature Discovery

The `features` array enumerates supported message types. A minimal provider need only support `["resolve"]`. The full set is:

| Feature | Required | Description |
|---------|----------|-------------|
| `resolve` | REQUIRED | Credential resolution |
| `lease` | RECOMMENDED | Lease renewal and release |
| `revoke` | RECOMMENDED | Emergency revocation |
| `list` | OPTIONAL | Discover available services/scopes |

---

## 6. Message Types

All messages follow JSON-RPC 2.0. The `method` field uses the `crp/` namespace.

### 6.1 `crp/resolve` — Request a Credential

The primary operation. A consumer requests a credential for a named service.

#### Request

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "method": "crp/resolve",
  "params": {
    "service": "github.com/api",
    "scopes": ["repo:read", "repo:write"],
    "credentialType": "oauth_bearer",
    "ttl": 3600,
    "context": {
      "consumer": "mcp-github-server",
      "consumerVersion": "2.1.0",
      "requestor": "agent:claude-desktop",
      "purpose": "List repository contents",
      "traceId": "abc-123-def"
    }
  }
}
```

#### Request Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `service` | string | REQUIRED | Namespaced service identifier |
| `scopes` | string[] | OPTIONAL | Requested permission scopes. Provider MAY narrow these. |
| `credentialType` | string | OPTIONAL | Preferred credential type. Provider MAY return a different type. |
| `ttl` | integer | OPTIONAL | Requested lease duration in seconds. Provider MAY shorten. |
| `context` | object | RECOMMENDED | Metadata for policy evaluation and audit. |
| `context.consumer` | string | RECOMMENDED | Name of the MCP server requesting the credential. |
| `context.consumerVersion` | string | OPTIONAL | Version of the consuming server. |
| `context.requestor` | string | OPTIONAL | Identity of the upstream client/agent. |
| `context.purpose` | string | OPTIONAL | Human-readable description of intended use. |
| `context.traceId` | string | OPTIONAL | Distributed tracing identifier. |

#### Response (Success)

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "result": {
    "credential": {
      "type": "oauth_bearer",
      "token": "ghp_xxxxxxxxxxxxxxxxxxxx"
    },
    "lease": {
      "id": "lease-7f3a9b2c",
      "expiresAt": "2026-02-14T00:50:00Z",
      "ttl": 3600,
      "renewable": true
    },
    "service": "github.com/api",
    "grantedScopes": ["repo:read"],
    "metadata": {
      "provider": "sanctum-local",
      "resolvedAt": "2026-02-13T23:50:00Z"
    }
  }
}
```

#### Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `credential` | object | REQUIRED | The resolved credential. Structure varies by type (see §7). |
| `credential.type` | string | REQUIRED | The credential type returned. |
| `lease` | object | REQUIRED | Lease information. |
| `lease.id` | string | REQUIRED | Opaque lease identifier for renewal/release. |
| `lease.expiresAt` | string | REQUIRED | ISO 8601 expiration timestamp. |
| `lease.ttl` | integer | REQUIRED | Lease duration in seconds. |
| `lease.renewable` | boolean | REQUIRED | Whether this lease can be renewed. |
| `service` | string | REQUIRED | Echoed service identifier. |
| `grantedScopes` | string[] | OPTIONAL | Scopes actually granted (may be subset of requested). |
| `metadata` | object | OPTIONAL | Provider metadata. |

---

### 6.2 `crp/lease` — Manage a Lease

Renew an active lease or release it early.

#### Renew Request

```json
{
  "jsonrpc": "2.0",
  "id": "req-002",
  "method": "crp/lease",
  "params": {
    "action": "renew",
    "leaseId": "lease-7f3a9b2c",
    "ttl": 3600
  }
}
```

#### Renew Response

```json
{
  "jsonrpc": "2.0",
  "id": "req-002",
  "result": {
    "lease": {
      "id": "lease-7f3a9b2c",
      "expiresAt": "2026-02-14T01:50:00Z",
      "ttl": 3600,
      "renewable": true
    },
    "credential": {
      "type": "oauth_bearer",
      "token": "ghp_yyyyyyyyyyyyyyyyyyyy"
    }
  }
}
```

Note: The `credential` field in the renew response is OPTIONAL. Providers MAY return a rotated credential on renewal, or omit it if the original credential remains valid.

#### Release Request

```json
{
  "jsonrpc": "2.0",
  "id": "req-003",
  "method": "crp/lease",
  "params": {
    "action": "release",
    "leaseId": "lease-7f3a9b2c"
  }
}
```

#### Release Response

```json
{
  "jsonrpc": "2.0",
  "id": "req-003",
  "result": {
    "released": true,
    "leaseId": "lease-7f3a9b2c"
  }
}
```

#### Lease Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | REQUIRED | `"renew"` or `"release"` |
| `leaseId` | string | REQUIRED | The lease to act on |
| `ttl` | integer | OPTIONAL | Requested renewal duration (renew only) |

---

### 6.3 `crp/revoke` — Emergency Revocation

Immediately invalidate a credential. Used when a compromise is detected.

#### Request

```json
{
  "jsonrpc": "2.0",
  "id": "req-004",
  "method": "crp/revoke",
  "params": {
    "leaseId": "lease-7f3a9b2c",
    "reason": "credential_compromised",
    "detail": "Token appeared in application logs"
  }
}
```

#### Response

```json
{
  "jsonrpc": "2.0",
  "id": "req-004",
  "result": {
    "revoked": true,
    "leaseId": "lease-7f3a9b2c",
    "revokedAt": "2026-02-13T23:55:00Z",
    "affectedLeases": ["lease-7f3a9b2c", "lease-9a1c3d4e"]
  }
}
```

#### Revoke Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `leaseId` | string | REQUIRED | Lease to revoke |
| `reason` | string | REQUIRED | One of: `credential_compromised`, `policy_violation`, `manual_revocation`, `rotation` |
| `detail` | string | OPTIONAL | Human-readable explanation |

#### Revoke Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `revoked` | boolean | REQUIRED | Whether revocation succeeded |
| `leaseId` | string | REQUIRED | Echoed lease ID |
| `revokedAt` | string | REQUIRED | ISO 8601 revocation timestamp |
| `affectedLeases` | string[] | OPTIONAL | Other leases invalidated (e.g., same underlying credential) |

---

### 6.4 `crp/list` — Discover Available Services

Query what services and scopes are available to the consumer.

#### Request

```json
{
  "jsonrpc": "2.0",
  "id": "req-005",
  "method": "crp/list",
  "params": {
    "filter": {
      "servicePrefix": "github.com/",
      "credentialType": "oauth_bearer"
    },
    "context": {
      "consumer": "mcp-github-server"
    }
  }
}
```

#### Response

```json
{
  "jsonrpc": "2.0",
  "id": "req-005",
  "result": {
    "services": [
      {
        "service": "github.com/api",
        "credentialTypes": ["oauth_bearer", "api_key"],
        "availableScopes": ["repo:read", "repo:write", "issues:read", "issues:write"],
        "maxTtl": 7200,
        "description": "GitHub REST and GraphQL API"
      },
      {
        "service": "github.com/packages",
        "credentialTypes": ["oauth_bearer"],
        "availableScopes": ["packages:read", "packages:write"],
        "maxTtl": 3600,
        "description": "GitHub Packages registry"
      }
    ],
    "cursor": null
  }
}
```

#### List Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `filter` | object | OPTIONAL | Narrow results |
| `filter.servicePrefix` | string | OPTIONAL | Service name prefix match |
| `filter.credentialType` | string | OPTIONAL | Filter by credential type |
| `context` | object | OPTIONAL | Used for policy-scoped listing |
| `cursor` | string | OPTIONAL | Pagination cursor from previous response |
| `limit` | integer | OPTIONAL | Max results per page (default: 50) |

#### List Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `services` | array | REQUIRED | Available service descriptors |
| `services[].service` | string | REQUIRED | Service identifier |
| `services[].credentialTypes` | string[] | REQUIRED | Supported credential types |
| `services[].availableScopes` | string[] | OPTIONAL | Scopes available for this service |
| `services[].maxTtl` | integer | OPTIONAL | Maximum lease duration in seconds |
| `services[].description` | string | OPTIONAL | Human-readable description |
| `cursor` | string | OPTIONAL | Pagination cursor, `null` if no more results |

---

## 7. Credential Types

CRP defines five standard credential types. Providers MAY support additional custom types.

### 7.1 `api_key`

A static API key or token.

```json
{
  "type": "api_key",
  "key": "sk-xxxxxxxxxxxxxxxxxxxxxxxx",
  "header": "Authorization",
  "prefix": "Bearer"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"api_key"` |
| `key` | string | REQUIRED | The key value |
| `header` | string | OPTIONAL | HTTP header name (default: `"Authorization"`) |
| `prefix` | string | OPTIONAL | Header value prefix (default: `"Bearer"`) |

### 7.2 `oauth_bearer`

An OAuth 2.x bearer token.

```json
{
  "type": "oauth_bearer",
  "token": "eyJhbGciOiJSUzI1NiIs...",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"oauth_bearer"` |
| `token` | string | REQUIRED | The bearer token |
| `tokenType` | string | OPTIONAL | Token type (default: `"Bearer"`) |
| `expiresIn` | integer | OPTIONAL | Token's own expiry in seconds (may differ from lease TTL) |

### 7.3 `client_certificate`

A TLS client certificate and private key.

```json
{
  "type": "client_certificate",
  "certificate": "-----BEGIN CERTIFICATE-----\n...",
  "privateKey": "-----BEGIN PRIVATE KEY-----\n...",
  "chain": ["-----BEGIN CERTIFICATE-----\n..."],
  "passphrase": null
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"client_certificate"` |
| `certificate` | string | REQUIRED | PEM-encoded certificate |
| `privateKey` | string | REQUIRED | PEM-encoded private key |
| `chain` | string[] | OPTIONAL | Intermediate CA certificates |
| `passphrase` | string | OPTIONAL | Private key passphrase, if encrypted |

### 7.4 `connection_string`

A database or service connection string.

```json
{
  "type": "connection_string",
  "value": "postgresql://user:pass@host:5432/db?sslmode=require",
  "dialect": "postgresql"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"connection_string"` |
| `value` | string | REQUIRED | The full connection string |
| `dialect` | string | OPTIONAL | Hint for the connection type (e.g., `postgresql`, `mysql`, `redis`, `mongodb`) |

### 7.5 `custom`

Extensible type for credentials not covered by the standard types.

```json
{
  "type": "custom",
  "customType": "aws_session",
  "data": {
    "accessKeyId": "AKIA...",
    "secretAccessKey": "...",
    "sessionToken": "...",
    "region": "us-east-1"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"custom"` |
| `customType` | string | REQUIRED | Provider-defined type name |
| `data` | object | REQUIRED | Arbitrary credential data |

---

## 8. Lease Model

Leases are the core security mechanism of CRP. Every resolved credential is bound to a lease.

### 8.1 Principles

1. **All credentials are leased.** There are no "permanent" credential grants.
2. **Providers set the TTL.** Consumers may request a TTL; providers MAY shorten it but MUST NOT exceed the requested TTL or their own maximum.
3. **Short leases are preferred.** Providers SHOULD default to the shortest TTL practical for the use case.
4. **Expiry is hard.** When a lease expires, the consumer MUST stop using the credential immediately. Providers SHOULD invalidate the credential if possible.

### 8.2 TTL Ranges

| Category | Recommended TTL | Max TTL |
|----------|----------------|---------|
| Interactive (user-driven) | 300–900s (5–15 min) | 3600s (1 hour) |
| Batch jobs | 900–3600s (15–60 min) | 14400s (4 hours) |
| Long-running services | 3600–7200s (1–2 hours) | 86400s (24 hours) |

These are recommendations. Providers enforce their own policies.

### 8.3 Renewal

Consumers SHOULD renew leases before expiry if they still need the credential. Providers MAY:

- Extend the existing lease (same credential, new expiry)
- Rotate the credential (new credential, new lease)
- Deny renewal (policy change, credential rotation required)

Consumers SHOULD attempt renewal at **50% of TTL** (e.g., renew a 1-hour lease at 30 minutes). This provides a buffer for transient failures.

### 8.4 Expiry Behavior

When a lease expires without renewal:

1. The provider MUST mark the lease as expired in its records.
2. The provider SHOULD invalidate or rotate the underlying credential if it was dynamically generated.
3. The provider MUST log the expiry event.
4. The consumer MUST discard the credential from memory.
5. The consumer MUST NOT attempt to use an expired credential.

### 8.5 Grace Period

Providers MAY implement a grace period (RECOMMENDED: 30 seconds) during which an expired lease can still be renewed. This accommodates clock skew and network latency. The grace period MUST NOT exceed 60 seconds.

---

## 9. Policy Integration

CRP providers enforce access policies. The protocol does not mandate a specific policy language but defines the integration points.

### 9.1 Policy Evaluation Points

Providers MUST evaluate policy at these points:

1. **On resolve** — Is this consumer authorized to access this service with these scopes?
2. **On renew** — Is the consumer still authorized? Has the policy changed?
3. **On list** — Only return services the consumer is authorized to see.

### 9.2 Policy Inputs

The `context` object in requests provides policy inputs. Providers SHOULD support decisions based on:

| Input | Source | Example Policy |
|-------|--------|---------------|
| Consumer identity | `context.consumer` | "Only `mcp-github-server` may access `github.com/api`" |
| Requestor identity | `context.requestor` | "Agent `claude-desktop` may access read scopes only" |
| Scopes | `params.scopes` | "No consumer gets `admin:*` scopes" |
| Time of day | System clock | "Production credentials only during business hours" |
| Rate | Provider tracking | "Max 10 resolutions per minute per consumer" |

### 9.3 Policy Responses

When policy denies a request, the provider MUST return an error (see §12) with sufficient detail for the consumer to understand the denial without revealing policy internals.

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "error": {
    "code": -33003,
    "message": "Policy denied",
    "data": {
      "service": "github.com/api",
      "deniedScopes": ["admin:org"],
      "reason": "Scope 'admin:org' is not permitted for this consumer"
    }
  }
}
```

---

## 10. Audit Requirements

Audit logging is non-negotiable for credential management. CRP providers MUST maintain audit logs.

### 10.1 Mandatory Audit Events

Providers MUST log the following events:

| Event | Trigger | Required Fields |
|-------|---------|-----------------|
| `credential.resolved` | Successful `crp/resolve` | timestamp, service, consumer, scopes granted, lease ID, TTL |
| `credential.denied` | Failed `crp/resolve` (policy) | timestamp, service, consumer, scopes requested, denial reason |
| `credential.error` | Failed `crp/resolve` (error) | timestamp, service, consumer, error code |
| `lease.renewed` | Successful `crp/lease` renew | timestamp, lease ID, new TTL |
| `lease.released` | Successful `crp/lease` release | timestamp, lease ID |
| `lease.expired` | Lease TTL elapsed | timestamp, lease ID, service |
| `credential.revoked` | Successful `crp/revoke` | timestamp, lease ID, reason, affected leases |

### 10.2 Prohibited Log Content

Providers MUST NOT log:

- Credential values (tokens, keys, passwords, certificates, connection strings)
- Private key material
- Any data sufficient to reconstruct or replay a credential

### 10.3 Log Retention

Providers SHOULD retain audit logs for a minimum of 90 days. Providers MUST document their retention policy.

### 10.4 Log Format

CRP does not mandate a log format. Providers SHOULD use structured logging (JSON) and SHOULD support export to common formats (syslog, OpenTelemetry, SIEM systems).

---

## 11. Security Considerations

### 11.1 Credential Handling

- **Memory.** Consumers MUST clear credential material from memory when the lease expires or is released. Languages with garbage collection SHOULD overwrite credential buffers where possible.
- **No logging.** Consumers MUST NOT log credential values. Providers MUST NOT log credential values. This applies to application logs, debug output, error messages, and crash reports.
- **No persistence.** Consumers MUST NOT write credentials to disk. The provider is the sole custodian of credential storage.
- **Transport.** When CRP messages traverse a network, TLS 1.2+ is REQUIRED. In-process and local stdio transports are exempt.

### 11.2 Lease Security

- **Lease IDs are secrets.** Treat lease IDs as bearer tokens. Possession of a lease ID allows renewal and (depending on provider) credential re-retrieval.
- **Lease ID entropy.** Providers MUST generate lease IDs with at least 128 bits of cryptographic randomness.
- **One consumer per lease.** Leases MUST NOT be shared across consumers.

### 11.3 Provider Security

- **Authentication.** Remote CRP providers MUST authenticate consumers. Recommended: mutual TLS or pre-shared tokens.
- **Authorization.** Providers MUST enforce policy (§9) on every request. Default-deny is REQUIRED.
- **Rate limiting.** Providers SHOULD implement rate limiting to prevent credential enumeration and denial-of-service.
- **Isolation.** Providers MUST isolate credential stores per tenant in multi-tenant deployments.

### 11.4 Threat Model

| Threat | Mitigation |
|--------|-----------|
| Credential leakage via logs | Mandatory no-log policy (§10.2, §11.1) |
| Stale credentials used after compromise | Short TTLs + `crp/revoke` |
| Unauthorized scope escalation | Policy enforcement on every resolve (§9) |
| Replay of lease IDs | High-entropy IDs + transport encryption |
| Provider compromise | Defense in depth: encrypted storage, HSM support, key rotation |
| Consumer impersonation | Consumer authentication (§11.3) |

---

## 12. Error Codes

CRP uses JSON-RPC 2.0 error codes. Standard JSON-RPC codes (-32700 to -32600) apply for parse/transport errors. CRP-specific errors use the -33xxx range.

| Code | Name | Description |
|------|------|-------------|
| -33001 | `version_mismatch` | CRP version incompatible |
| -33002 | `service_not_found` | Requested service identifier is unknown |
| -33003 | `policy_denied` | Policy evaluation denied the request |
| -33004 | `lease_not_found` | Lease ID does not exist or has expired |
| -33005 | `lease_expired` | Lease has expired and cannot be renewed |
| -33006 | `lease_not_renewable` | Lease does not support renewal |
| -33007 | `credential_unavailable` | Credential exists but cannot be resolved (e.g., upstream service down) |
| -33008 | `rate_limited` | Too many requests; retry after delay |
| -33009 | `scope_unavailable` | Requested scope does not exist for this service |
| -33010 | `provider_error` | Internal provider error |
| -33011 | `feature_not_supported` | Requested CRP feature not supported by this provider |
| -33012 | `revocation_failed` | Revocation could not be completed |

### Error Response Format

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "error": {
    "code": -33008,
    "message": "Rate limited",
    "data": {
      "retryAfter": 30,
      "limit": "10 requests per minute",
      "service": "github.com/api"
    }
  }
}
```

The `data` field is OPTIONAL but RECOMMENDED. It MUST NOT contain credential material.

---

## 13. Reference Implementation

### 13.1 SanctumAI

[SanctumAI](https://sanctumai.dev) provides the reference implementation of CRP v0.1. SanctumAI is a local-first credential vault for AI agents—the "SQLite of agent credential management."

**Provider type:** In-process library + stdio subprocess  
**Storage backend:** Encrypted local vault (SQLCipher)  
**Policy engine:** Declarative TOML-based policies  
**Audit:** Structured JSON logs to local file + optional remote export  

SanctumAI implements all four CRP message types (`resolve`, `lease`, `revoke`, `list`) and all five credential types.

### 13.2 Implementing a CRP Provider

A minimal CRP provider requires:

1. **Message handler** — Parse JSON-RPC, route `crp/resolve` requests.
2. **Credential store** — Any backend: environment variables, encrypted file, cloud KMS.
3. **Lease tracker** — In-memory map of lease ID → expiry. Clean up expired leases periodically.
4. **Policy check** — Even a simple allowlist satisfies the requirement.
5. **Audit log** — Append-only log file with the mandatory events from §10.1.

A basic provider in ~200 lines of Python or TypeScript is realistic. The protocol is intentionally simple to minimize implementation barriers.

### 13.3 Conformance Levels

| Level | Requirements | Use Case |
|-------|-------------|----------|
| **Basic** | `crp/resolve` + lease tracking + audit logging | Personal/dev use |
| **Standard** | Basic + `crp/lease` + `crp/revoke` + policy enforcement | Production single-tenant |
| **Full** | Standard + `crp/list` + multi-tenant isolation + rate limiting | Hosted/enterprise |

---

## 14. IANA/Registry Considerations

### 14.1 Current Status

CRP v0.1 operates within MCP's `experimental` namespace. No IANA registration is required.

### 14.2 Future Standardization Path

If CRP is adopted into MCP core, the following registrations would be proposed:

1. **Method namespace:** `crp/` methods promoted from `experimental.crp` to a first-class MCP capability.
2. **Error code range:** -33001 to -33099 reserved for CRP.
3. **Credential type registry:** A maintained list of standard credential type identifiers.
4. **Service identifier conventions:** Guidance on service naming (reverse-domain recommended: `github.com/api`).

### 14.3 Extension Namespace

Third-party extensions to CRP SHOULD use the `x-crp/` method prefix and `x-` prefixed credential types to avoid collisions with future standard types.

---

## 15. Appendix: Example Flows

### 15.1 Basic API Key Resolution

An MCP server for OpenAI needs an API key.

**Step 1: Resolve**

```json
// Consumer → Provider
{
  "jsonrpc": "2.0",
  "id": "1",
  "method": "crp/resolve",
  "params": {
    "service": "openai.com/api",
    "credentialType": "api_key",
    "ttl": 900,
    "context": {
      "consumer": "mcp-openai-server",
      "requestor": "agent:cursor",
      "purpose": "Generate embeddings for RAG pipeline"
    }
  }
}
```

```json
// Provider → Consumer
{
  "jsonrpc": "2.0",
  "id": "1",
  "result": {
    "credential": {
      "type": "api_key",
      "key": "sk-proj-xxxxxxxxxxxx",
      "header": "Authorization",
      "prefix": "Bearer"
    },
    "lease": {
      "id": "lease-a1b2c3d4",
      "expiresAt": "2026-02-14T00:05:00Z",
      "ttl": 900,
      "renewable": true
    },
    "service": "openai.com/api",
    "grantedScopes": ["embeddings", "chat.completions"],
    "metadata": {
      "provider": "sanctum-local",
      "resolvedAt": "2026-02-13T23:50:00Z"
    }
  }
}
```

**Step 2: Use the credential (outside CRP scope)**

```
POST https://api.openai.com/v1/embeddings
Authorization: Bearer sk-proj-xxxxxxxxxxxx
```

**Step 3: Release when done**

```json
// Consumer → Provider
{
  "jsonrpc": "2.0",
  "id": "2",
  "method": "crp/lease",
  "params": {
    "action": "release",
    "leaseId": "lease-a1b2c3d4"
  }
}
```

```json
// Provider → Consumer
{
  "jsonrpc": "2.0",
  "id": "2",
  "result": {
    "released": true,
    "leaseId": "lease-a1b2c3d4"
  }
}
```

---

### 15.2 OAuth Token with Renewal

An MCP server maintains a long-running connection to GitHub.

**Step 1: Initial resolve**

```json
// Consumer → Provider
{
  "jsonrpc": "2.0",
  "id": "10",
  "method": "crp/resolve",
  "params": {
    "service": "github.com/api",
    "scopes": ["repo:read"],
    "credentialType": "oauth_bearer",
    "ttl": 1800,
    "context": {
      "consumer": "mcp-github-server",
      "requestor": "agent:claude-desktop"
    }
  }
}
```

```json
// Provider → Consumer
{
  "jsonrpc": "2.0",
  "id": "10",
  "result": {
    "credential": {
      "type": "oauth_bearer",
      "token": "gho_abcdef123456",
      "tokenType": "Bearer",
      "expiresIn": 1800
    },
    "lease": {
      "id": "lease-gh-001",
      "expiresAt": "2026-02-14T00:20:00Z",
      "ttl": 1800,
      "renewable": true
    },
    "service": "github.com/api",
    "grantedScopes": ["repo:read"]
  }
}
```

**Step 2: Renew at 50% TTL (15 minutes later)**

```json
// Consumer → Provider
{
  "jsonrpc": "2.0",
  "id": "11",
  "method": "crp/lease",
  "params": {
    "action": "renew",
    "leaseId": "lease-gh-001",
    "ttl": 1800
  }
}
```

```json
// Provider → Consumer (with rotated token)
{
  "jsonrpc": "2.0",
  "id": "11",
  "result": {
    "lease": {
      "id": "lease-gh-001",
      "expiresAt": "2026-02-14T00:50:00Z",
      "ttl": 1800,
      "renewable": true
    },
    "credential": {
      "type": "oauth_bearer",
      "token": "gho_rotated789xyz",
      "tokenType": "Bearer",
      "expiresIn": 1800
    }
  }
}
```

---

### 15.3 Emergency Revocation

A credential appears in error logs. The consumer (or an admin tool) triggers emergency revocation.

```json
// Consumer → Provider
{
  "jsonrpc": "2.0",
  "id": "99",
  "method": "crp/revoke",
  "params": {
    "leaseId": "lease-gh-001",
    "reason": "credential_compromised",
    "detail": "OAuth token found in Sentry error report"
  }
}
```

```json
// Provider → Consumer
{
  "jsonrpc": "2.0",
  "id": "99",
  "result": {
    "revoked": true,
    "leaseId": "lease-gh-001",
    "revokedAt": "2026-02-14T00:35:00Z",
    "affectedLeases": ["lease-gh-001"]
  }
}
```

The provider:
1. Invalidates lease `lease-gh-001`
2. Revokes the underlying OAuth token at GitHub's API (if supported)
3. Logs the revocation event with full context
4. Alerts any admin notification channels (provider-specific)

---

### 15.4 Service Discovery

A new MCP server starts up and wants to know what credentials are available.

```json
// Consumer → Provider
{
  "jsonrpc": "2.0",
  "id": "50",
  "method": "crp/list",
  "params": {
    "context": {
      "consumer": "mcp-multi-tool-server"
    }
  }
}
```

```json
// Provider → Consumer
{
  "jsonrpc": "2.0",
  "id": "50",
  "result": {
    "services": [
      {
        "service": "openai.com/api",
        "credentialTypes": ["api_key"],
        "availableScopes": ["chat.completions", "embeddings", "images", "audio"],
        "maxTtl": 3600,
        "description": "OpenAI API"
      },
      {
        "service": "github.com/api",
        "credentialTypes": ["oauth_bearer", "api_key"],
        "availableScopes": ["repo:read", "repo:write", "issues:read", "issues:write"],
        "maxTtl": 7200,
        "description": "GitHub REST and GraphQL API"
      },
      {
        "service": "postgres/analytics",
        "credentialTypes": ["connection_string"],
        "availableScopes": ["read"],
        "maxTtl": 900,
        "description": "Analytics database (read-only)"
      },
      {
        "service": "aws/s3",
        "credentialTypes": ["custom"],
        "availableScopes": ["s3:GetObject", "s3:PutObject"],
        "maxTtl": 3600,
        "description": "AWS S3 via session credentials"
      }
    ],
    "cursor": null
  }
}
```

---

## Appendix A: JSON Schema Definitions

### A.1 CRP Resolve Request Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://crp.dev/schema/v0.1/resolve-request.json",
  "title": "CRP Resolve Request",
  "type": "object",
  "required": ["jsonrpc", "id", "method", "params"],
  "properties": {
    "jsonrpc": { "const": "2.0" },
    "id": { "type": ["string", "integer"] },
    "method": { "const": "crp/resolve" },
    "params": {
      "type": "object",
      "required": ["service"],
      "properties": {
        "service": {
          "type": "string",
          "description": "Namespaced service identifier",
          "pattern": "^[a-zA-Z0-9._/-]+$"
        },
        "scopes": {
          "type": "array",
          "items": { "type": "string" }
        },
        "credentialType": {
          "type": "string",
          "enum": ["api_key", "oauth_bearer", "client_certificate", "connection_string", "custom"]
        },
        "ttl": {
          "type": "integer",
          "minimum": 1,
          "maximum": 86400
        },
        "context": {
          "type": "object",
          "properties": {
            "consumer": { "type": "string" },
            "consumerVersion": { "type": "string" },
            "requestor": { "type": "string" },
            "purpose": { "type": "string" },
            "traceId": { "type": "string" }
          }
        }
      }
    }
  }
}
```

### A.2 CRP Resolve Response Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://crp.dev/schema/v0.1/resolve-response.json",
  "title": "CRP Resolve Response",
  "type": "object",
  "required": ["jsonrpc", "id", "result"],
  "properties": {
    "jsonrpc": { "const": "2.0" },
    "id": { "type": ["string", "integer"] },
    "result": {
      "type": "object",
      "required": ["credential", "lease", "service"],
      "properties": {
        "credential": {
          "type": "object",
          "required": ["type"],
          "properties": {
            "type": {
              "type": "string",
              "enum": ["api_key", "oauth_bearer", "client_certificate", "connection_string", "custom"]
            }
          }
        },
        "lease": {
          "type": "object",
          "required": ["id", "expiresAt", "ttl", "renewable"],
          "properties": {
            "id": { "type": "string" },
            "expiresAt": { "type": "string", "format": "date-time" },
            "ttl": { "type": "integer", "minimum": 1 },
            "renewable": { "type": "boolean" }
          }
        },
        "service": { "type": "string" },
        "grantedScopes": {
          "type": "array",
          "items": { "type": "string" }
        },
        "metadata": {
          "type": "object",
          "properties": {
            "provider": { "type": "string" },
            "resolvedAt": { "type": "string", "format": "date-time" }
          }
        }
      }
    }
  }
}
```

---

## Appendix B: Comparison with Existing Approaches

| Approach | Scope | CRP Advantage |
|----------|-------|---------------|
| Environment variables | Static, no rotation, no audit | Dynamic resolution, leasing, full audit |
| HashiCorp Vault | Full-featured but heavy, not MCP-aware | MCP-native, lightweight, agent-focused |
| AWS Secrets Manager | Cloud-locked, no local-first option | Provider-agnostic, local or remote |
| 1Password CLI | Consumer-focused, no lease model | Agent-native, lease-based, policy-aware |
| `.env` files | Zero security, zero audit | Everything `.env` lacks |

CRP is not a replacement for enterprise secret managers. It is the **protocol layer** that lets MCP servers talk to *any* credential source—including those enterprise systems—through a standard interface.

---

## Appendix C: Changelog

### v0.1 (2026-02-13) — Initial Draft

- Initial protocol definition
- Four message types: resolve, lease, revoke, list
- Five credential types: api_key, oauth_bearer, client_certificate, connection_string, custom
- Lease model with TTL, renewal, and expiry
- Policy integration framework
- Audit requirements
- Security considerations
- Error code registry (-33001 to -33012)
- Reference implementation: SanctumAI

---

*This document is a community draft. Feedback and contributions are welcome. The canonical version is maintained at [github.com/sanctumai/crp-spec](https://github.com/sanctumai/crp-spec).*
