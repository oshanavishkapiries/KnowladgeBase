# Zero Trust — Complete Study Guide

> An exhaustive guide to the Zero Trust security model: the philosophy, NIST 800-207 architecture, real tools (Cloudflare Access, Tailscale, Okta, BeyondCorp), and practical implementation roadmap.

---

## What Is This Guide?

Zero Trust is not a product — it is a **security philosophy and architectural strategy**. This guide covers the "never trust, always verify" model from its Google BeyondCorp origins to modern enterprise deployments using ZTNA, SASE, microsegmentation, and identity-aware access.

---

## Document Index

| # | File | What You Will Learn |
|---|------|---------------------|
| 01 | [Philosophy & Principles](./01-philosophy-and-principles.md) | The "never trust" model, BeyondCorp history, why VPNs fail |
| 02 | [NIST 800-207 Architecture](./02-nist-800-207-architecture.md) | PDP, PEP, trust algorithm, logical components |
| 03 | [Identity & Access Management](./03-identity-and-access-management.md) | MFA, device posture, context-aware access, IAM |
| 04 | [Microsegmentation & Networks](./04-microsegmentation-and-networks.md) | East-west traffic, SDN, service mesh, SASE |
| 05 | [Tools & Platforms](./05-tools-and-platforms.md) | Cloudflare Access, Tailscale, Okta, Google BeyondCorp, Pomerium |
| 06 | [Implementation Roadmap](./06-implementation-roadmap.md) | NIST 5-phase model, migration from VPN, pilot strategy |
| 07 | [Edge Cases & Failure Modes](./07-edge-cases-failure-modes.md) | Common pitfalls, bypass attacks, misconfiguration risks |

---

## The Zero Trust Mindset Shift

```
OLD MODEL (Castle-and-Moat):
  ┌─────────────────────────────────────┐
  │         Corporate Network           │
  │  ┌──────────────────────────────┐  │
  │  │  "Trusted" Internal Network  │  │
  │  │  Once inside → full access   │  │
  │  └──────────────────────────────┘  │
  └─────────────────────────────────────┘
           ▲ Firewall perimeter

ZERO TRUST MODEL:
  Everything is untrusted — inside or outside.
  
  User + Device ──▶ Identity Verification ──▶ Device Posture Check
                                                      │
                                                      ▼
                                            Policy Engine Decision
                                                      │
                                              ┌───────┴────────┐
                                              │                │
                                            ALLOW           DENY
                                              │
                                              ▼
                                    Specific Resource Access
                                    (Least Privilege, Time-bound)
```

---

## Core Tenets (CISA + NIST)

1. **All data sources and computing services are resources** — no implicit trust based on network location
2. **All communication is secured** — regardless of network location (TLS everywhere)
3. **Access to individual resources is granted per-session** — not persistent
4. **Access is determined by dynamic policy** — identity + device posture + context
5. **All assets are monitored for security posture** — continuous verification
6. **Authentication and authorization are dynamic** — continuously re-evaluated
7. **Collect security data to improve posture** — logs, telemetry, behavioral analytics

---

## Key Resources

- NIST SP 800-207: https://nvlpubs.nist.gov/nistpubs/specialpublications/NIST.SP.800-207.pdf
- CISA Zero Trust Maturity Model: https://www.cisa.gov/zero-trust-maturity-model
- Google BeyondCorp Research Papers: https://research.google/pubs/?area=security-privacy-and-abuse-prevention
- Cloudflare Zero Trust: https://developers.cloudflare.com/cloudflare-one/
- Tailscale Docs: https://tailscale.com/kb/
