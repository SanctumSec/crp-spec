# Changelog

## 0.2.0 — 2026-07-17 (Proposed)

- **`crp/use` is the REQUIRED primary operation** — returns lease + `proxyBaseUrl`, never a secret.
- Protocol is **use-not-retrieve only**: no raw secret export method in CRP.
- Delegation with bounded depth and cascade revocation.
- Brokered / federated credentials (e.g. Entra Agent ID, OAuth2 client-credentials).
- Classification descriptors (`public` … `restricted`).
- Graduated outcomes: `require_approval` (`-33020`), `quarantine` (`-33021`).
- Conformance tiers rewritten around `crp/use`.
- Capability `features` list centers on `use` (no resolve).

## 0.1.0 — 2026-02-13 (Superseded)

- Initial draft. Primary operation was `crp/resolve` (return credential material).
- Superseded by 0.2.0 for security practice (use without holding).
