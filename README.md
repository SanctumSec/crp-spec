# Credential Resolution Protocol (CRP)

**The missing security layer for AI agents.**

CRP is an open protocol specification for how AI agents and MCP servers resolve credentials needed to access downstream APIs and services. It addresses the "last mile" credential problem — the gap between authenticating *to* a tool server and authenticating *from* that server to the services it calls on your behalf.

## The Problem

MCP's OAuth 2.1 flow handles client → server authentication. But when an MCP server needs to call OpenAI, Stripe, GitHub, or any downstream API on behalf of an agent, there's no standard for how those credentials are requested, delivered, scoped, or audited.

Today's reality: API keys hardcoded in environment variables, `.env` files committed to repos, secrets scattered across agent configs with zero policy enforcement. This doesn't scale. It's not secure. It's not auditable.

## The Solution

CRP defines four simple primitives:

| Method | Purpose | Required |
|--------|---------|----------|
| `crp/resolve` | Request a credential for a named service | ✅ Yes |
| `crp/list` | Discover available credential scopes | No |
| `crp/lease` | Extend or release a credential lease | No |
| `crp/revoke` | Emergency credential revocation | No |

Every credential is delivered with a **lease** — a time-bounded grant that expires automatically. No permanent credential exposure. Every resolution is **auditable**. Every request passes through **policy checks**.

## Conformance Levels

| Level | Requirements | Effort |
|-------|-------------|--------|
| **Basic** | `crp/resolve` + leases | A weekend |
| **Standard** | + `crp/list` + `crp/lease` + audit logging | A week |
| **Full** | + `crp/revoke` + policy integration + anomaly detection | Production-grade |

## Integration with MCP

CRP integrates via MCP's `experimental` capability — no spec fork required:

```json
{
  "capabilities": {
    "experimental": {
      "crp": {
        "version": "0.1",
        "provider": "sanctum"
      }
    }
  }
}
```

## Quick Example

**Agent requests an API key:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "crp/resolve",
  "params": {
    "service": "openai",
    "credentialType": "api_key",
    "scopes": ["chat.completions"],
    "context": {
      "consumer": "coding-assistant-v2",
      "purpose": "Generate code completion"
    }
  }
}
```

**Provider responds with a leased credential:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "credential": {
      "type": "api_key",
      "value": "sk-...",
      "service": "openai"
    },
    "lease": {
      "id": "lease-abc123",
      "expiresAt": "2026-02-13T23:30:00Z",
      "renewable": true
    }
  }
}
```

## Specification

📄 **[Read the full spec →](spec.md)**

## Reference Implementation

[SanctumAI](https://sanctumai.dev) provides the reference implementation of CRP, built on a local-first credential vault with policy enforcement, audit logging, and anomaly detection.

**SDKs available:** [Rust](https://crates.io/crates/sanctum-ai) · [Python](https://pypi.org/project/sanctum-ai/) · [Node.js](https://www.npmjs.com/package/sanctum-ai) · [Go](https://github.com/SanctumSec/sanctum-sdk-go)

## Contributing

CRP is an open standard. We welcome contributions, feedback, and alternative implementations.

- **Spec issues:** Open an issue in this repo
- **Discussion:** [MCP Working Group](https://github.com/modelcontextprotocol/specification/discussions)
- **Community demand:** [Issue #754](https://github.com/modelcontextprotocol/servers/issues/754) · [Discussion #220](https://github.com/modelcontextprotocol/specification/discussions/220)

## License

This specification is released under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/). Implement freely. Attribute kindly.

---

**Website:** [crp.dev](https://crp.dev) · **Protocol by:** [SanctumAI](https://sanctumai.dev)
