# Caddy — The Ultimate Server with Automatic HTTPS

> A complete, in-depth study guide for mastering Caddy: the modern, zero-config web server written in Go.

---

## What Is This Guide?

This guide covers Caddy from first principles to production-grade deployments. It is structured to serve both developers new to Caddy and experienced engineers who want to understand its internals, performance characteristics, and failure modes.

---

## Document Index

| # | File | What You Will Learn |
|---|------|---------------------|
| 01 | [Introduction & Architecture](./01-introduction-and-architecture.md) | What Caddy is, Go internals, module system, config lifecycle |
| 02 | [Automatic HTTPS Deep Dive](./02-automatic-https-deep-dive.md) | ACME protocol, Let's Encrypt, ZeroSSL, local CA, OCSP, cert renewal |
| 03 | [Caddyfile Configuration](./03-caddyfile-configuration.md) | Syntax, site blocks, directives, matchers, config adapters |
| 04 | [Reverse Proxy & Load Balancing](./04-reverse-proxy-and-load-balancing.md) | Upstreams, health checks, load balancing strategies, headers |
| 05 | [HTTP/3 & Modern Protocols](./05-http3-quic-modern-protocols.md) | QUIC, HTTP/3, Alt-Svc, 0-RTT, WebTransport |
| 06 | [Performance & Benchmarks](./06-performance-and-benchmarks.md) | Caddy vs Nginx vs Apache, memory, concurrency, tuning |
| 07 | [Plugins & Module System](./07-plugins-and-modules.md) | Writing custom modules, xcaddy, community plugins |
| 08 | [Real-World Deployment](./08-real-world-deployment.md) | Docker, Kubernetes, systemd, zero-downtime reloads |
| 09 | [Edge Cases & Failure Modes](./09-edge-cases-and-failure-modes.md) | Rate limits, cert failures, split-brain, debugging |

---

## Quick Mental Model

```
Caddyfile / JSON / YAML / TOML / NGINX config
        │
        ▼  (Config Adapter)
    JSON (native config)
        │
        ▼  (Admin API → Config Loading)
   ┌────────────────────────────────┐
   │          Caddy Core            │
   │  ┌────────┐  ┌──────────────┐ │
   │  │  HTTP  │  │     TLS      │ │
   │  │  App   │  │   Manager    │ │
   │  └────────┘  └──────────────┘ │
   │  ┌────────┐  ┌──────────────┐ │
   │  │  PKI   │  │   Storage    │ │
   │  │  App   │  │  (certmagic) │ │
   │  └────────┘  └──────────────┘ │
   └────────────────────────────────┘
```

---

## Prerequisites

- Basic understanding of HTTP/HTTPS
- Familiarity with the command line
- Optional: Go knowledge (for plugin development)

---

## Key Resources

- Official Docs: https://caddyserver.com/docs
- GitHub: https://github.com/caddyserver/caddy
- Community Forum: https://caddy.community
- Plugin Directory: https://caddyserver.com/download
