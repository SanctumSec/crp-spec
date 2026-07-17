# Credential Resolution Protocol (CRP)

**The last-mile credential standard for MCP.**

CRP is an open protocol for how MCP servers **use** credentials for downstream APIs **without the secret ever leaving a trusted provider**. It fills the gap OAuth 2.1 leaves open: client → server is secured; server → GitHub/OpenAI/DBs still often means plaintext `.env` keys.

## Status

| | |
|--|--|
| **Version** | **0.2.0** (Proposed) |
| **Primary operation** | `crp/use` (REQUIRED) — lease + proxy handle, never a secret |
| **Target** | MCP Working Group (`experimental` capability) |
| **Spec** | [`spec.md`](./spec.md) |

## The problem

```
Client ──(OAuth 2.1)──▶ MCP Server ──( ??? )──▶ Downstream API
                              raw keys in env
```

CRP standardizes the `???`:

```
Client ──(OAuth 2.1)──▶ MCP Server ──(crp/use)──▶ Provider ──inject in transit──▶ Downstream
```

## Core idea: use, don't retrieve

```json
// Request
{ "method": "crp/use", "params": { "service": "github.com/api", "ttl": 3600 } }

// Response — no secret
{
  "lease": { "id": "l-123", "expiresAt": "...", "ttl": 3600 },
  "proxyBaseUrl": "http://localhost:7700/api/v1/proxy/t/l-123"
}
```

Point your SDK at `proxyBaseUrl`. The provider injects the credential in transit.

## Operations (v0.2)

| Method | Role |
|--------|------|
| **`crp/use`** | **REQUIRED** — obtain use-handle (lease + proxy) |
| `crp/list` | OPTIONAL — discover services (no secrets) |
| `crp/lease` | RECOMMENDED — renew / release |
| `crp/revoke` | RECOMMENDED — kill-switch (+ cascade on delegated leases) |

Plus: delegation, brokered federated identity, classification, graduated outcomes (`require_approval`, `quarantine`).

CRP does **not** define returning raw credential material to the consumer.

## MCP integration

```json
{
  "experimental": {
    "crp": {
      "version": "0.2",
      "provider": "example-vault",
      "features": ["use", "list", "lease", "revoke", "delegate", "broker", "classify"],
      "conformance": "standard"
    }
  }
}
```

## Conformance tiers

| Tier | Focus |
|------|--------|
| **Basic** | `crp/use` + leases + audit |
| **Standard** | + list/revoke + version negotiation + classification |
| **Full** | + delegation, brokering, graduated outcomes, policy |

## Reference implementation

[Sanctum](https://www.sanctumai.dev) is an optional local vault that implements the use-not-retrieve model. CRP is provider-agnostic — any vault can conform.

## Site

Protocol site (when deployed): [crp.dev](https://www.crp.dev)

## License

[CC-BY-4.0](./LICENSE)

## Authors

Jason Gale (SanctumAI)
