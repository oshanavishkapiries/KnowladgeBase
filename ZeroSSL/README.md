# ZeroSSL — Complete Study Guide

> An in-depth guide to ZeroSSL: a modern Certificate Authority offering free and paid TLS certificates via ACME, REST API, and a visual dashboard.

---

## What Is This Guide?

This guide covers everything about ZeroSSL — how it works as a CA, how the ACME protocol integrates with it, how to use its REST API for automation, how it differs from Let's Encrypt, and how to integrate it with real tools.

---

## Document Index

| # | File | What You Will Learn |
|---|------|---------------------|
| 01 | [What Is ZeroSSL](./01-what-is-zerossl.md) | History, what ZeroSSL offers, how it fits into the CA ecosystem |
| 02 | [ACME Protocol & EAB](./02-acme-protocol-and-eab.md) | How ACME works with ZeroSSL, External Account Binding, challenges |
| 03 | [REST API & Management](./03-rest-api-and-management.md) | Programmatic cert management without ACME |
| 04 | [Certificate Types & Pricing](./04-certificate-types-and-pricing.md) | Free tier, 90-day, annual, wildcard, OV, EV comparison |
| 05 | [Integration Guide](./05-integration-guide.md) | Caddy, Certbot, acme.sh, cert-manager, Docker |
| 06 | [Edge Cases & Troubleshooting](./06-edge-cases-troubleshooting.md) | EAB errors, rate limits, validation failures, debugging |

---

## Quick Mental Model

```
ZeroSSL sits in the CA ecosystem alongside Let's Encrypt and Buypass:

Client (Certbot / Caddy / acme.sh)
        │
        │  ACME Protocol (RFC 8555)
        ▼
  ZeroSSL ACME Directory
  https://acme.zerossl.com/v2/DV90
        │
        │  EAB (External Account Binding) required
        │  (unlike Let's Encrypt which doesn't require EAB)
        ▼
  ZeroSSL Certificate Authority
        │
        │  Issues DV (Domain Validated) TLS Certificate
        ▼
  Your Server (Nginx / Caddy / Apache / etc.)
```

---

## Key Differentiators vs Let's Encrypt

| Feature | ZeroSSL | Let's Encrypt |
|---------|---------|---------------|
| Free ACME certs | ✅ Unlimited | ✅ Unlimited |
| EAB required | ✅ Yes | ❌ No |
| REST API | ✅ Yes | ❌ No |
| Web dashboard | ✅ Yes | ❌ No |
| Wildcard (free) | ✅ ACME | ✅ ACME |
| 1-year certs | ✅ Paid | ❌ No |
| OV / EV certs | ✅ Paid | ❌ No |
| Rate limits (ACME) | Higher | 50/domain/week |
| Caddy built-in | ✅ Native support | ✅ Native support |

---

## Key Resources

- Official Docs: https://zerossl.com/documentation/
- ACME Endpoint: `https://acme.zerossl.com/v2/DV90`
- REST API Docs: https://zerossl.com/documentation/api/
- Dashboard: https://app.zerossl.com
