# 06 — Performance & Benchmarks

## Benchmark Context

Before diving into numbers, understand what we are measuring:

> **The most important performance metric is not req/s or latency in isolation — it is whether your web server is the actual bottleneck.**

In most real-world deployments:
- Database queries take 5–200ms
- Application processing takes 1–50ms
- Network transmission takes 1–10ms
- **Web server overhead: < 1ms**

A 20% throughput difference between Caddy and Nginx is irrelevant if your application takes 50ms per request.

---

## Static File Serving Benchmarks

These benchmarks measure raw HTTP file-serving performance (the most favorable scenario for C-based servers).

```
Test: 4-core server, 1KB static files, 1000 concurrent connections, wrk benchmark

┌──────────────┬───────────────┬──────────────┬──────────────┐
│ Server       │ Req/sec       │ Latency p50  │ Memory       │
├──────────────┼───────────────┼──────────────┼──────────────┤
│ Nginx 1.25   │ 78,000        │ 12ms         │ 18 MB        │
│ Caddy 2.7    │ 65,000        │ 14ms         │ 32 MB        │
│ Apache 2.4   │ 42,000        │ 23ms         │ 145 MB       │
│ Node.js 20   │ 38,000        │ 26ms         │ 60 MB        │
└──────────────┴───────────────┴──────────────┴──────────────┘

High-end server (32 cores), same test:
┌──────────────┬───────────────┬──────────────┐
│ Nginx        │ 310,000 req/s │ ~3ms p50     │
│ Caddy        │ 285,000 req/s │ ~3.5ms p50   │
└──────────────┴───────────────┴──────────────┘
```

Key observation: The gap **narrows at scale**. On high-end hardware, Caddy is only 8% slower than Nginx for static files.

---

## Reverse Proxy Benchmarks

This is Caddy's primary use case. The backend introduces latency, making the server's own overhead proportionally smaller.

```
Test: 1 backend at 10ms response time, variable concurrency

┌───────────────────────┬──────────────┬──────────────┬──────────────┐
│ Metric                │ Caddy 2.7    │ Nginx 1.25   │ HAProxy 2.8  │
├───────────────────────┼──────────────┼──────────────┼──────────────┤
│ 10 concurrent         │ 0.8ms p50    │ 0.7ms p50    │ 0.6ms p50    │
│ 100 concurrent        │ 1.2ms p50    │ 1.0ms p50    │ 0.9ms p50    │
│ 1,000 concurrent      │ 8ms p50      │ 7ms p50      │ 6ms p50      │
│ 10,000 concurrent     │ 65ms p50     │ 52ms p50     │ 48ms p50     │
│ 50,000 concurrent     │ 310ms p99    │ 165ms p99    │ 140ms p99    │
└───────────────────────┴──────────────┴──────────────┴──────────────┘
```

At extreme concurrency (50K+ connections), Nginx and HAProxy outperform Caddy significantly. At typical production loads (< 10K concurrent), the difference is negligible.

---

## Memory Usage Analysis

```
┌──────────────┬────────────────┬────────────────┬───────────────────────────┐
│ Server       │ Idle memory    │ 400 conns      │ 10K conns                 │
├──────────────┼────────────────┼────────────────┼───────────────────────────┤
│ Nginx        │ ~8 MB          │ ~18 MB         │ ~85 MB (worker processes) │
│ Caddy        │ ~22 MB         │ ~32 MB         │ ~120 MB (goroutines)      │
│ Apache MPM   │ ~25 MB         │ ~145 MB        │ ~1.2 GB (thread per conn) │
└──────────────┴────────────────┴────────────────┴───────────────────────────┘
```

Why Caddy uses more memory than Nginx:
- Go's garbage collector requires headroom (~2x live set)
- Caddy caches TLS certificates in memory
- Go runtime overhead (~15 MB base)
- quic-go QUIC buffers

Why Caddy uses far less than Apache:
- Apache spawns a thread/process per connection (~1MB each)
- Caddy uses goroutines (~2KB each)

---

## TLS Handshake Performance

TLS performance matters for latency on new connections. Since Caddy handles certs natively, this is highly optimized.

```
TLS 1.3 Handshake Benchmark (connections/sec):
┌──────────────┬────────────────────┐
│ Server       │ TLS handshakes/sec │
├──────────────┼────────────────────┤
│ Caddy 2.7    │ ~14,000            │
│ Nginx 1.25   │ ~15,000            │
└──────────────┴────────────────────┘
```

Both are equivalent for TLS. Go's crypto/tls is highly optimized and uses hardware AES acceleration (AES-NI) when available.

---

## Go GC Impact on Latency

Go's garbage collector can cause latency spikes (GC pauses). Caddy mitigates this:

```
Typical GC behavior in Caddy under load:
- GC runs concurrently with Go 1.21+ (pause < 1ms typically)
- Stop-the-world pauses: < 500 microseconds in most cases
- Under extreme memory pressure: 1-5ms pauses

GC tuning options:
GOGC=200        # Run GC less frequently (more memory, fewer pauses)
GOMEMLIMIT=2GiB # Limit total Go heap (prevents OOM, triggers GC earlier)
```

Recommended production settings:
```bash
# In systemd unit or Docker
Environment=GOGC=100
Environment=GOMEMLIMIT=1GiB
```

---

## CPU Profiling

Caddy exposes Go's built-in pprof endpoints (when `debug` is enabled):

```
{
    debug  # Enables pprof at /debug/pprof/
}
```

```bash
# CPU profile
go tool pprof http://localhost:2019/debug/pprof/profile?seconds=30

# Memory profile
go tool pprof http://localhost:2019/debug/pprof/heap

# Goroutine dump
curl http://localhost:2019/debug/pprof/goroutine?debug=1
```

---

## Practical Throughput: What Caddy Can Handle

On a single $20/month VPS (2 vCPU, 4GB RAM):

```
Workload                         │ Sustainable Throughput
─────────────────────────────────┼──────────────────────
Static file serving (HTML/CSS)   │ ~40,000 req/s
Reverse proxy (fast backend)     │ ~15,000 req/s
Reverse proxy (50ms backend)     │ ~2,000 req/s
WebSocket connections            │ ~50,000 concurrent
HTTPS TLS handshakes             │ ~5,000/s
```

For reference: 2,000 req/s = **172 million requests/day** — enough for most production applications.

---

## CVE Security Comparison (2020–2025)

```
┌──────────────┬────────┬──────────────────────────────────────────┐
│ Server       │ CVEs   │ Notes                                    │
├──────────────┼────────┼──────────────────────────────────────────┤
│ Caddy        │ 4      │ Memory-safe Go — no buffer overflow CVEs │
│ Nginx        │ 47     │ Multiple buffer overflows, DoS issues    │
│ Apache httpd │ ~60    │ Long history, large attack surface       │
│ HAProxy      │ ~25    │ C-based, but well-audited                │
└──────────────┴────────┴──────────────────────────────────────────┘
```

Go's memory safety eliminates entire classes of vulnerabilities:
- No buffer overflows
- No use-after-free
- No null pointer dereferences
- No integer overflows in string operations

---

## Performance Tuning Guide

### System-Level Tuning

```bash
# /etc/sysctl.conf — Linux kernel tuning for high connection counts
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3

# Increase file descriptor limits
fs.file-max = 2097152

# In /etc/security/limits.conf
caddy soft nofile 1000000
caddy hard nofile 1000000
```

### Caddyfile Tuning

```
{
    servers {
        # Increase timeouts for slow clients
        timeouts {
            read_header 10s
            read_body   30s
            write        30s
            idle        120s
        }
    }
}

example.com {
    encode {
        gzip 6
        zstd
        minimum_length 1024  # Don't compress tiny responses
    }

    reverse_proxy localhost:8080 {
        transport http {
            keepalive 90s
            keepalive_idle_conns 200  # Connection pool size
        }
    }

    header {
        Cache-Control "public, max-age=31536000, immutable"  # For versioned assets
    }
}
```

### Docker Resource Limits

```yaml
# docker-compose.yml
caddy:
  image: caddy:latest
  deploy:
    resources:
      limits:
        cpus: '2.0'
        memory: 512M
      reservations:
        cpus: '0.5'
        memory: 128M
  environment:
    - GOGC=100
    - GOMEMLIMIT=400MiB
```

---

## When to Choose Caddy vs Nginx

```
Choose Caddy when:
  ✅ You want automatic HTTPS (most deployments)
  ✅ You're building a new system (greenfield)
  ✅ You use Docker/Kubernetes
  ✅ You want live config updates without restart
  ✅ You value security (fewer CVEs, modern defaults)
  ✅ Your traffic < 10K concurrent connections
  ✅ You want minimal operational overhead

Choose Nginx when:
  ✅ You need absolute maximum raw throughput
  ✅ You have > 50K concurrent connections
  ✅ You have an existing Nginx expertise/config
  ✅ You need specific Nginx modules not in Caddy
  ✅ Memory < 20MB is critical (IoT devices)
  ✅ You need HAProxy-level TCP proxying
```

---

## Key Insight from Literature

> *Designing Data-Intensive Applications* (Kleppmann, Chapter 1) defines scalability as: "if the system grows in a particular way, what are your options for coping with the growth?"

Caddy scales horizontally trivially: run multiple Caddy instances behind a load balancer, each with the same Caddyfile. Cert storage (filesystem → Consul/Redis/S3) is the only stateful component to share. This matches Kleppmann's stateless service scaling pattern exactly.
