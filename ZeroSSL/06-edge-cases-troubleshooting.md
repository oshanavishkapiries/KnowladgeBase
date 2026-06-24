# 06 — Edge Cases & Troubleshooting

## Common Failure Categories

```
1. EAB Credential Failures     — Wrong KID/HMAC, expired credentials
2. Domain Validation Failures  — Challenge not reachable, DNS not propagated
3. Rate Limit Errors           — Too many requests, free tier limit hit
4. Certificate Storage Issues  — Corrupted files, permission errors
5. Client Configuration Errors — Wrong directory URL, missing flags
6. Renewal Failures            — Cron not running, changed validation setup
```

---

## 1. EAB Credential Failures

### Error: "External account binding required"

```
Error message:
  urn:ietf:params:acme:error:externalAccountRequired
  "External account binding is required for this CA"

Cause: You're using a standard ACME client config without EAB,
       pointing at ZeroSSL's directory URL.

Fix: Add EAB credentials to your ACME client config.

Certbot:
  certbot certonly \
    --server https://acme.zerossl.com/v2/DV90 \
    --eab-kid "YOUR_KID" \        ← add this
    --eab-hmac-key "YOUR_HMAC" \  ← add this
    ...

acme.sh:
  acme.sh --register-account \
    --server zerossl \
    --eab-kid "YOUR_KID" \
    --eab-hmac-key "YOUR_HMAC"
```

### Error: "Invalid EAB credentials"

```
Error message:
  urn:ietf:params:acme:error:unauthorized
  "The provided EAB HMAC key is invalid"

Causes:
  1. HMAC key copied with extra whitespace or newline
  2. Base64 encoding issue (URL-safe vs standard Base64)
  3. KID and HMAC from different accounts

Diagnosis:
  # Check your EAB values
  echo -n "YOUR_HMAC" | base64 -d | wc -c  # Should be 32 bytes for HS256
  
Fix:
  1. Re-generate EAB credentials from dashboard
  2. Copy-paste carefully — no extra spaces
  3. Store in environment variables, not inline:
     export EAB_KID="$(cat /secrets/zerossl-kid)"
     export EAB_HMAC="$(cat /secrets/zerossl-hmac)"
```

### Error: "EAB account already registered"

```
Cause: You're trying to create a new ACME account with EAB credentials
       that are already bound to an existing ACME account.

Fix (acme.sh): 
  # Remove old account and re-register
  rm ~/.acme.sh/ca/acme.zerossl.com/
  acme.sh --register-account --server zerossl --eab-kid ... --eab-hmac-key ...

Fix (Certbot):
  # Remove account
  sudo rm -rf /etc/letsencrypt/accounts/acme.zerossl.com
  # Re-run certbot with --register-unsafely-without-email or with --email
```

---

## 2. Domain Validation Failures

### HTTP-01 Challenge Failure

```
Error:
  "Challenge failed for domain example.com. 
   Type: http-01. Error: Connection refused"

Diagnosis steps:
  # 1. Test the challenge URL manually
  curl -v http://example.com/.well-known/acme-challenge/test
  
  # 2. Check if port 80 is open
  sudo ss -tlnp | grep ':80'
  sudo ufw status | grep '80'
  
  # 3. Test from an external perspective
  curl -v --resolve example.com:80:YOUR_SERVER_IP http://example.com/.well-known/acme-challenge/test

Common causes and fixes:
  ├── Port 80 blocked by firewall → open port 80 (needed for challenge only)
  ├── Web server not running → start nginx/apache/caddy
  ├── Another server handles port 80 → configure it to proxy challenge path
  ├── Cloudflare proxy enabled → either disable "orange cloud" or use DNS challenge
  └── Server behind NAT → ensure port forwarding for port 80
```

### DNS Propagation Failure (DNS-01 Challenge)

```
Error:
  "DNS problem: NXDOMAIN looking up TXT for _acme-challenge.example.com"

Diagnosis:
  # Check if TXT record exists
  dig TXT _acme-challenge.example.com @8.8.8.8
  dig TXT _acme-challenge.example.com @1.1.1.1
  
  # If not found, check your DNS provider panel

Causes:
  1. API credentials wrong (Cloudflare token missing DNS:Edit permission)
  2. DNS propagation hasn't completed (wait up to 60 seconds, TTL matters)
  3. Wrong zone — multiple zones with same domain name
  4. Record created in wrong account

For Cloudflare API token:
  Required permissions:
    Zone → Zone → Read
    Zone → DNS → Edit
  
  # Test token
  curl -X GET "https://api.cloudflare.com/client/v4/user/tokens/verify" \
    -H "Authorization: Bearer YOUR_TOKEN"
    
# Increase propagation wait in certbot
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-propagation-seconds 60 \  ← increase this
  ...
```

### Challenge Token Mismatch

```
Error:
  "Response did not match expected keyAuthorization"

Cause: Multiple Certbot/acme.sh instances running,
       or challenge file was overwritten by another process.

Fix:
  # Check for other acme processes
  ps aux | grep -E 'certbot|acme|caddy'
  
  # Kill conflicting processes
  sudo killall certbot
  
  # Try again with only one ACME client
```

---

## 3. Rate Limit Errors

### REST API Free Tier Limit (3 certs)

```
Error (HTTP 400):
  "You have reached your certificate limit"

Context: Free tier dashboard/REST API allows max 3 ACTIVE certs.

Solutions:
  Option A: Use ACME protocol instead (unlimited)
    → Switch to Certbot/acme.sh/Caddy with ZeroSSL ACME endpoint

  Option B: Revoke/delete old certificates you no longer need
    → Dashboard: "Certificates" → select → Revoke
    → API: DELETE /certificates/{id}

  Option C: Upgrade to paid plan
    → zerossl.com/pricing
```

### ACME Rate Limit

```
Error:
  "urn:ietf:params:acme:error:rateLimited"
  "You have exceeded a rate limit."

ZeroSSL ACME limits are higher than Let's Encrypt but not unlimited.

Short-term fix: Wait a few hours before retrying
Long-term fix:
  - Batch domain certificates (use SAN certs for multiple domains)
  - Use wildcard certs where possible
  - Ensure renewal happens only when needed (not repeated)
```

---

## 4. Certificate Storage Issues

### Corrupted Certificate File

```
Error (Nginx): "SSL_CTX_use_certificate_file() failed"
Error (Caddy): "failed to load certificate: PEM decode failed"

Diagnosis:
  # Verify certificate is valid PEM
  openssl x509 -in certificate.crt -noout -text
  
  # Check certificate chain
  openssl verify -CAfile ca_bundle.crt certificate.crt
  
  # Check if cert matches private key
  openssl x509 -noout -modulus -in certificate.crt | md5sum
  openssl rsa -noout -modulus -in private.key | md5sum
  # Both hashes MUST match

Fix:
  # Re-download certificate from ZeroSSL
  curl "https://api.zerossl.com/certificates/{CERT_ID}/download/return?access_key=$KEY" > cert.json
  jq -r '."certificate.crt"' cert.json > certificate.crt
  jq -r '."ca_bundle.crt"' cert.json > ca_bundle.crt
  cat certificate.crt ca_bundle.crt > fullchain.crt
```

### Permission Issues

```
Error: "Permission denied: /etc/ssl/private/example.com.key"

Fix:
  # Key file should be 600 (owner-read only)
  sudo chmod 600 /etc/ssl/private/example.com.key
  sudo chown root:root /etc/ssl/private/example.com.key
  
  # Or for web server user (e.g., www-data)
  sudo chown www-data:www-data /etc/ssl/private/example.com.key
  sudo chmod 640 /etc/ssl/private/example.com.key
```

---

## 5. Client Configuration Errors

### Certbot Pointing to Wrong Server

```
Error: "This account is not authorized to perform this challenge"

Cause: Account was created with Let's Encrypt but you're trying 
       to use ZeroSSL (or vice versa). ACME accounts are CA-specific.

Fix:
  # Check current accounts
  sudo ls /etc/letsencrypt/accounts/
  
  # For ZeroSSL: accounts live under acme.zerossl.com/
  sudo ls /etc/letsencrypt/accounts/acme.zerossl.com/
  
  # For Let's Encrypt: accounts live under acme-v02.api.letsencrypt.org/
  sudo ls /etc/letsencrypt/accounts/acme-v02.api.letsencrypt.org/
  
  # Always specify --server explicitly to avoid confusion
  certbot certonly --server https://acme.zerossl.com/v2/DV90 ...
```

---

## 6. Renewal Failures

### Certbot Auto-Renewal Not Working

```
Diagnosis:
  # Test renewal
  sudo certbot renew --dry-run --server https://acme.zerossl.com/v2/DV90
  
  # Check cron job
  crontab -l | grep certbot
  sudo crontab -l | grep certbot
  cat /etc/cron.d/certbot
  
  # Check systemd timer (modern systems)
  sudo systemctl list-timers | grep certbot
  sudo systemctl status certbot.timer
  sudo systemctl status certbot.service
  
  # Check logs
  sudo journalctl -u certbot
  sudo cat /var/log/letsencrypt/letsencrypt.log

Common fixes:
  # If cron is disabled:
  sudo systemctl enable --now certbot.timer
  
  # If renewal hook is broken:
  sudo certbot renew --deploy-hook "systemctl reload nginx"
```

### Certificate Expired Despite Auto-Renewal

```
This happens when:
  1. Renewal ran but web server wasn't reloaded
  2. Renewal failed silently (check logs!)
  3. Certificate path in config doesn't match renewed path

Prevention:
  # Always use --deploy-hook for post-renewal action
  certbot certonly \
    --deploy-hook "systemctl reload nginx" \
    ...
  
  # Or add to /etc/letsencrypt/renewal/example.com.conf:
  [renewalparams]
  renew_hook = systemctl reload nginx
  
  # Monitor cert expiry (add to crontab)
  0 8 * * * /usr/local/bin/check-cert-expiry.sh | mail -s "Cert Check" admin@example.com
```

---

## Debugging Toolkit

```bash
#!/bin/bash
# zerossl-debug.sh — comprehensive ZeroSSL debugging

DOMAIN="${1:-example.com}"
API_KEY="${ZEROSSL_API_KEY:-not-set}"

echo "=== ZeroSSL Debug Report for $DOMAIN ==="
echo ""

echo "--- DNS Resolution ---"
dig A "$DOMAIN" +short
dig AAAA "$DOMAIN" +short

echo ""
echo "--- Current Certificate ---"
echo | openssl s_client -connect "$DOMAIN:443" -servername "$DOMAIN" 2>/dev/null | \
  openssl x509 -noout -text 2>/dev/null | \
  grep -E "Subject:|Issuer:|Not Before|Not After"

echo ""
echo "--- Certificate Expiry ---"
EXPIRY=$(echo | openssl s_client -connect "$DOMAIN:443" -servername "$DOMAIN" 2>/dev/null | \
  openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
echo "Expires: $EXPIRY"

echo ""
echo "--- OCSP Stapling ---"
echo | openssl s_client -connect "$DOMAIN:443" -status -servername "$DOMAIN" 2>/dev/null | \
  grep -A 6 "OCSP Response"

echo ""
echo "--- Port 80 Reachability ---"
curl -sv "http://$DOMAIN/.well-known/acme-challenge/test" 2>&1 | \
  grep -E "Connected|Response|curl:"

if [ "$API_KEY" != "not-set" ]; then
  echo ""
  echo "--- ZeroSSL API Status ---"
  curl -s "https://api.zerossl.com/certificates?access_key=$API_KEY&search=$DOMAIN" | \
    jq '.results[] | {domain: .common_name, status: .status, expires: .expiration}'
fi
```

---

## Quick Reference: Error → Fix Table

```
┌──────────────────────────────────────────┬──────────────────────────────────────────┐
│ Error                                    │ Fix                                      │
├──────────────────────────────────────────┼──────────────────────────────────────────┤
│ externalAccountRequired                  │ Add --eab-kid and --eab-hmac-key         │
│ unauthorized (invalid EAB)               │ Re-generate EAB from dashboard            │
│ Connection refused (HTTP-01)             │ Open port 80 in firewall                 │
│ NXDOMAIN (_acme-challenge)               │ Wait for DNS propagation; check API key  │
│ Certificate limit reached                │ Use ACME (unlimited) or upgrade plan     │
│ SSL_CTX_use_certificate_file failed      │ Verify cert is valid PEM; re-download    │
│ Permission denied (key file)             │ chmod 600 on key file                    │
│ Cert expired despite renewal             │ Add --deploy-hook for server reload       │
│ Response did not match keyAuthorization  │ Kill other ACME clients; retry            │
│ rateLimited                              │ Wait a few hours; use SAN/wildcard certs │
└──────────────────────────────────────────┴──────────────────────────────────────────┘
```
