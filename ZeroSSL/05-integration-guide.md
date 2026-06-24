# 05 — ZeroSSL Integration Guide

## Integration Overview

ZeroSSL works with every major ACME client and web server. The integration pattern is always:

```
1. Get EAB credentials from ZeroSSL dashboard or API
2. Configure ACME client with ZeroSSL's directory URL + EAB
3. Run certificate automation as you would with Let's Encrypt
```

---

## Caddy Integration (Best Developer Experience)

Caddy has **first-class native support** for ZeroSSL — including automatic EAB credential generation. No manual dashboard visit required.

### Method 1: Caddy Auto-Configuration (Zero Manual Steps)

```
# Caddyfile — Caddy uses both LE and ZeroSSL automatically
{
    email admin@example.com  # Caddy auto-generates ZeroSSL EAB for you
}

example.com {
    reverse_proxy localhost:3000
}
```

Caddy calls `https://api.zerossl.com/acme/eab-credentials` using your email, gets EAB credentials, then uses ZeroSSL ACME. **No dashboard visit, no manual EAB setup.**

### Method 2: Force ZeroSSL Only

```
# Caddyfile — use only ZeroSSL (skip Let's Encrypt)
{
    email admin@example.com
}

example.com {
    tls {
        issuer zerossl {
            email admin@example.com
        }
    }
    reverse_proxy localhost:3000
}
```

### Method 3: Explicit EAB (JSON config)

```json
{
  "apps": {
    "tls": {
      "automation": {
        "policies": [
          {
            "subjects": ["example.com"],
            "issuers": [
              {
                "module": "zerossl",
                "email": "admin@example.com",
                "eab": {
                  "key_id": "YOUR_EAB_KID",
                  "mac_key": "YOUR_EAB_HMAC_KEY"
                }
              }
            ]
          }
        ]
      }
    }
  }
}
```

---

## Certbot Integration

Certbot requires manual EAB setup (no auto-generation like Caddy).

### Prerequisites

```bash
# Install Certbot
sudo apt install certbot

# Get EAB credentials from https://app.zerossl.com/developer
export EAB_KID="your-eab-kid"
export EAB_HMAC="your-eab-hmac-key"
export DOMAIN="example.com"
export EMAIL="admin@example.com"
```

### Standalone Mode (Port 80)

```bash
# Stop your web server first (Certbot will use port 80)
sudo systemctl stop nginx

sudo certbot certonly \
  --standalone \
  --server https://acme.zerossl.com/v2/DV90 \
  --eab-kid "$EAB_KID" \
  --eab-hmac-key "$EAB_HMAC" \
  --email "$EMAIL" \
  --agree-tos \
  --no-eff-email \
  -d "$DOMAIN"

# Certificate location:
# /etc/letsencrypt/live/example.com/fullchain.pem
# /etc/letsencrypt/live/example.com/privkey.pem
```

### Webroot Mode (No Downtime)

```bash
# Keep web server running — Certbot serves challenge via webroot
sudo certbot certonly \
  --webroot \
  --webroot-path /var/www/html \
  --server https://acme.zerossl.com/v2/DV90 \
  --eab-kid "$EAB_KID" \
  --eab-hmac-key "$EAB_HMAC" \
  --email "$EMAIL" \
  --agree-tos \
  -d "$DOMAIN" \
  -d "www.$DOMAIN"
```

### DNS Challenge (Wildcard)

```bash
# Install Certbot DNS plugin for Cloudflare
sudo pip3 install certbot-dns-cloudflare

# Create Cloudflare credentials file
cat > ~/.secrets/cloudflare.ini << EOF
dns_cloudflare_api_token = YOUR_CF_TOKEN
EOF
chmod 600 ~/.secrets/cloudflare.ini

# Obtain wildcard cert
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials ~/.secrets/cloudflare.ini \
  --server https://acme.zerossl.com/v2/DV90 \
  --eab-kid "$EAB_KID" \
  --eab-hmac-key "$EAB_HMAC" \
  --email "$EMAIL" \
  --agree-tos \
  -d "*.example.com" \
  -d "example.com"

# Setup auto-renewal (Certbot adds cron job automatically)
sudo certbot renew --dry-run
```

---

## acme.sh Integration

acme.sh is a lightweight alternative to Certbot, written in pure shell.

```bash
# Install acme.sh
curl https://get.acme.sh | sh -s email=admin@example.com

# Register account with ZeroSSL EAB
~/.acme.sh/acme.sh \
  --register-account \
  --server zerossl \
  --eab-kid "YOUR_EAB_KID" \
  --eab-hmac-key "YOUR_EAB_HMAC"

# Issue certificate (HTTP validation)
~/.acme.sh/acme.sh \
  --issue \
  --server zerossl \
  -d example.com \
  --webroot /var/www/html

# Issue wildcard (DNS validation with Cloudflare)
export CF_Token="YOUR_CF_TOKEN"
~/.acme.sh/acme.sh \
  --issue \
  --server zerossl \
  -d "*.example.com" \
  -d "example.com" \
  --dns dns_cf

# Install certificate to Nginx
~/.acme.sh/acme.sh \
  --install-cert -d example.com \
  --cert-file /etc/nginx/ssl/example.com.crt \
  --key-file /etc/nginx/ssl/example.com.key \
  --fullchain-file /etc/nginx/ssl/example.com.fullchain.crt \
  --reloadcmd "systemctl reload nginx"

# acme.sh adds renewal cron automatically
crontab -l | grep acme
```

---

## Nginx Integration

```bash
# After obtaining cert via Certbot or acme.sh
```

```nginx
# /etc/nginx/sites-available/example.com

server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # Certificates from ZeroSSL (via Certbot)
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # Modern TLS configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    resolver 8.8.8.8 1.1.1.1 valid=300s;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    
    # Security headers
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Docker Integration

### Option 1: Caddy in Docker (Easiest)

```yaml
# docker-compose.yml
version: "3.9"

services:
  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - ACME_EMAIL=admin@example.com

  app:
    image: my-app:latest
    restart: unless-stopped

volumes:
  caddy_data:
  caddy_config:
```

```
# Caddyfile — uses ZeroSSL automatically
{
    email {$ACME_EMAIL}
}

example.com {
    reverse_proxy app:3000
}
```

### Option 2: Nginx + acme.sh in Docker

```dockerfile
# Dockerfile.certbot
FROM certbot/certbot:latest

# Pre-configure ZeroSSL EAB
ENV ZEROSSL_EAB_KID=""
ENV ZEROSSL_EAB_HMAC=""
ENV DOMAIN=""
ENV EMAIL=""

ENTRYPOINT certbot certonly \
  --standalone \
  --server https://acme.zerossl.com/v2/DV90 \
  --eab-kid "$ZEROSSL_EAB_KID" \
  --eab-hmac-key "$ZEROSSL_EAB_HMAC" \
  --email "$EMAIL" \
  --agree-tos \
  -d "$DOMAIN"
```

```yaml
# docker-compose with separate cert fetcher
version: "3.9"
services:
  certbot:
    build:
      context: .
      dockerfile: Dockerfile.certbot
    volumes:
      - certbot_certs:/etc/letsencrypt
    environment:
      - ZEROSSL_EAB_KID=${EAB_KID}
      - ZEROSSL_EAB_HMAC=${EAB_HMAC}
      - DOMAIN=example.com
      - EMAIL=admin@example.com

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - certbot_certs:/etc/letsencrypt:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - certbot

volumes:
  certbot_certs:
```

---

## Kubernetes with cert-manager

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Verify
kubectl get pods -n cert-manager
```

```yaml
# zerossl-cluster-issuer.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: zerossl-eab
  namespace: cert-manager
type: Opaque
stringData:
  hmac: "YOUR_BASE64_HMAC_KEY"  # Put in k8s secret, not plaintext!

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: zerossl
spec:
  acme:
    server: https://acme.zerossl.com/v2/DV90
    email: admin@example.com
    externalAccountBinding:
      keyID: YOUR_EAB_KID
      keySecretRef:
        name: zerossl-eab
        key: hmac
    privateKeySecretRef:
      name: zerossl-account-key
    solvers:
      - http01:
          ingress:
            class: nginx

---
# Certificate resource
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: zerossl
    kind: ClusterIssuer
  dnsNames:
    - example.com
    - www.example.com
  duration: 2160h    # 90 days
  renewBefore: 720h  # Renew 30 days before expiry
```

```yaml
# Ingress using ZeroSSL cert
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    cert-manager.io/cluster-issuer: "zerossl"
spec:
  tls:
    - hosts:
        - example.com
      secretName: example-com-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

---

## Renewal Automation

For all ACME-based setups (Certbot, acme.sh), renewal is automatic via cron. For the REST API:

```bash
#!/bin/bash
# zerossl-renew.sh — check and renew via REST API if needed

API_KEY="${ZEROSSL_API_KEY}"
CERT_ID="${ZEROSSL_CERT_ID}"
DOMAIN="${DOMAIN}"

# Get cert expiry
EXPIRY=$(curl -s "https://api.zerossl.com/certificates/$CERT_ID?access_key=$API_KEY" | jq -r '.expiration')
EXPIRY_TS=$(date -d "$EXPIRY" +%s 2>/dev/null || gdate -d "$EXPIRY" +%s)
NOW_TS=$(date +%s)
DAYS_LEFT=$(( (EXPIRY_TS - NOW_TS) / 86400 ))

echo "Certificate expires in $DAYS_LEFT days"

if [ "$DAYS_LEFT" -lt 30 ]; then
    echo "Renewing certificate..."
    # Use acme.sh or certbot to renew
    ~/.acme.sh/acme.sh --renew -d "$DOMAIN" --server zerossl --force
    echo "Certificate renewed!"
fi
```

```bash
# Add to crontab (run daily at 3 AM)
0 3 * * * /path/to/zerossl-renew.sh >> /var/log/zerossl-renew.log 2>&1
```
