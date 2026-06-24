# 05 — Zero Trust Tools & Platforms

## Tool Landscape Overview

No single tool provides complete Zero Trust. The ecosystem is layered:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ LAYER                   │ FUNCTION                │ TOOLS               │
├─────────────────────────┼─────────────────────────┼─────────────────────┤
│ Identity Provider (IdP) │ Who is the user?        │ Okta, Azure AD,     │
│                         │                         │ Google, JumpCloud   │
├─────────────────────────┼─────────────────────────┼─────────────────────┤
│ Device Management (MDM) │ Is the device trusted?  │ Intune, Jamf,       │
│                         │                         │ CrowdStrike, Kandji │
├─────────────────────────┼─────────────────────────┼─────────────────────┤
│ Network Access (ZTNA)   │ What can they reach?    │ Cloudflare Access,  │
│                         │                         │ Tailscale, Zscaler  │
├─────────────────────────┼─────────────────────────┼─────────────────────┤
│ App-Level Proxy (IAP)   │ Identity before app     │ Pomerium, BeyondCorp│
│                         │                         │ Nginx+auth_request  │
├─────────────────────────┼─────────────────────────┼─────────────────────┤
│ Service Mesh            │ Service-to-service auth │ Istio, Linkerd,     │
│                         │                         │ Consul Connect      │
├─────────────────────────┼─────────────────────────┼─────────────────────┤
│ Secrets Management      │ Machine credentials     │ HashiCorp Vault,    │
│                         │                         │ AWS Secrets Mgr     │
├─────────────────────────┼─────────────────────────┼─────────────────────┤
│ Monitoring / SIEM       │ What's happening?       │ Splunk, Datadog,    │
│                         │                         │ Microsoft Sentinel  │
└─────────────────────────┴─────────────────────────┴─────────────────────┘
```

---

## Cloudflare Zero Trust

**What it is**: A complete SASE/ZTNA platform combining Cloudflare's global network with Zero Trust security controls.

**Components**:
- **Cloudflare Access**: Identity-aware proxy for web apps (replaces VPN for HTTP)
- **Cloudflare Tunnel**: Exposes internal apps without opening firewall ports
- **Cloudflare Gateway**: DNS filtering, Secure Web Gateway
- **Cloudflare WARP**: Device agent for network-level Zero Trust
- **Cloudflare DLP**: Data Loss Prevention

**Free tier**: Up to 50 users free — best free Zero Trust offering in 2026.

### Setting Up Cloudflare Access

```bash
# Step 1: Install cloudflared on your server
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
chmod +x cloudflared
sudo mv cloudflared /usr/local/bin/

# Step 2: Authenticate with Cloudflare
cloudflared tunnel login

# Step 3: Create a tunnel
cloudflared tunnel create my-app-tunnel
# Creates credentials at ~/.cloudflared/<TUNNEL_ID>.json

# Step 4: Configure the tunnel
cat > ~/.cloudflared/config.yml << EOF
tunnel: <TUNNEL_ID>
credentials-file: /home/user/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: internal-app.example.com
    service: http://localhost:3000
  - hostname: admin.example.com
    service: http://localhost:8080
  - service: http_status:404
EOF

# Step 5: Create DNS record
cloudflared tunnel route dns my-app-tunnel internal-app.example.com

# Step 6: Start the tunnel
cloudflared tunnel run my-app-tunnel

# Run as systemd service
sudo cloudflared service install
sudo systemctl start cloudflared
```

```yaml
# Cloudflare Access Policy (configured in Cloudflare Dashboard or via API)
# Protect internal-app.example.com

# Policy: "Engineering team during business hours"
Policy Name: "Engineering Access"
Decision: Allow

Include:
  - Emails ending in: @example.com
  - Identity provider group: "Engineering"

Require (AND logic):
  - Country: United States, Canada
  - Device posture: Managed device

Additional rules:
  - If accessing /admin/* → Require re-authentication
  - Block: Known malicious IPs (Cloudflare threat intelligence)
```

### Cloudflare Service Token (Machine-to-Machine)

For CI/CD pipelines and services:

```bash
# Create service token in Cloudflare Dashboard
# Dashboard → Access → Service Auth → Create Service Token

# Use in CI/CD (e.g., GitHub Actions)
- name: Deploy to production
  env:
    CF_ACCESS_CLIENT_ID: ${{ secrets.CF_ACCESS_CLIENT_ID }}
    CF_ACCESS_CLIENT_SECRET: ${{ secrets.CF_ACCESS_CLIENT_SECRET }}
  run: |
    curl -H "CF-Access-Client-Id: $CF_ACCESS_CLIENT_ID" \
         -H "CF-Access-Client-Secret: $CF_ACCESS_CLIENT_SECRET" \
         https://internal-api.example.com/deploy
```

---

## Tailscale

**What it is**: A WireGuard-based mesh VPN with Zero Trust access control. Each device gets a stable IP (100.x.x.x) and communicates directly (peer-to-peer when possible).

**Best for**: 
- Developer SSH access to servers
- Database access without VPN
- Service-to-service communication across clouds
- Small-medium teams

**Free tier**: Up to 3 users, 100 devices.

### Quick Start

```bash
# Install on any device (Linux, macOS, Windows, iOS, Android)
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Check your tailnet IP
tailscale ip
# → 100.64.1.5

# SSH to another tailnet device
ssh user@100.64.1.10

# Or use MagicDNS (device hostnames)
ssh user@my-server
```

### Access Control Lists (ACL)

Tailscale ACLs are defined in a JSON file (or HuJSON) in the admin console:

```json
{
  "groups": {
    "group:engineering": ["alice@example.com", "bob@example.com"],
    "group:dba": ["carol@example.com"],
    "group:devops": ["dave@example.com"]
  },

  "tagOwners": {
    "tag:prod-server": ["group:devops"],
    "tag:staging-server": ["group:engineering"],
    "tag:database": ["group:devops"]
  },

  "acls": [
    {
      "action": "accept",
      "src": ["group:engineering"],
      "dst": ["tag:staging-server:22", "tag:staging-server:8080"]
    },
    {
      "action": "accept",
      "src": ["group:dba"],
      "dst": ["tag:database:5432"]
    },
    {
      "action": "accept",
      "src": ["group:devops"],
      "dst": ["tag:prod-server:22", "tag:database:5432"]
    }
  ],

  "ssh": [
    {
      "action": "accept",
      "src": ["group:engineering"],
      "dst": ["tag:staging-server"],
      "users": ["ubuntu", "ec2-user"]
    },
    {
      "action": "accept",
      "src": ["group:devops"],
      "dst": ["tag:prod-server"],
      "users": ["ubuntu"],
      "checkPeriod": "8h"   # Re-verify auth every 8 hours
    }
  ]
}
```

### Tailscale Serve (Expose Local Services)

```bash
# Expose localhost:3000 as https://myhost.tail12345.ts.net
tailscale serve https / http://localhost:3000

# Expose a specific path
tailscale serve https /api http://localhost:8080

# Funnel (expose to public internet, not just tailnet)
tailscale funnel 443 on
```

---

## Google BeyondCorp Enterprise

**What it is**: Google's commercial product built on the same architecture they use internally.

**Components**:
- Identity-Aware Proxy (IAP): Protects apps with identity verification
- Chrome Enterprise Premium: Device posture via Chrome browser
- BeyondCorp Enterprise Threat and Data Protection: DLP, malware scanning
- Context-Aware Access: Dynamic policies based on context

```python
# Using Google IAP to protect a Python Flask app
from google.auth.transport.requests import Request
from google.oauth2 import id_token

def verify_iap_token(request):
    """Verify the IAP JWT token"""
    iap_jwt = request.headers.get("X-Goog-IAP-JWT-Assertion")
    if not iap_jwt:
        return None, "No IAP token"
    
    audience = "/projects/PROJECT_NUMBER/apps/PROJECT_ID"
    
    try:
        decoded = id_token.verify_token(
            iap_jwt,
            Request(),
            audience=audience,
            certs_url="https://www.gstatic.com/iap/verify/public_key"
        )
        return decoded, None
    except Exception as e:
        return None, str(e)

# In your Flask route
@app.route('/admin')
def admin():
    user_info, error = verify_iap_token(request)
    if error:
        return "Unauthorized", 403
    
    email = user_info.get("email")
    return f"Hello, {email}!"
```

```yaml
# GCP IAP Configuration via Terraform
resource "google_iap_web_backend_service_iam_binding" "binding" {
  project         = var.project_id
  web_backend_service = google_compute_backend_service.app.name
  role            = "roles/iap.httpsResourceAccessor"
  
  members = [
    "group:engineering@example.com",
    "user:alice@example.com",
  ]
  
  condition {
    title       = "Business hours access"
    description = "Allow access Monday-Friday 9AM-6PM"
    expression  = "request.time.getHours('America/New_York') >= 9 && request.time.getHours('America/New_York') < 18 && request.time.getDayOfWeek('America/New_York') >= 1 && request.time.getDayOfWeek('America/New_York') <= 5"
  }
}
```

---

## Okta (Identity Provider)

**What it is**: The leading standalone Identity Provider (IdP) for enterprise Zero Trust.

**Key capabilities**:
- SSO (Single Sign-On) for all apps via SAML/OIDC
- MFA enforcement (TOTP, push, FIDO2/WebAuthn)
- Adaptive authentication (risk-based step-up)
- SCIM for automated user provisioning/deprovisioning
- API Access Management (OAuth 2.0 authorization server)
- Device Trust integration with MDM providers

### Okta Zero Trust Policy Example

```javascript
// Okta Expression Language — adaptive MFA policy
// Allow access only if: 
//   - User is in Engineering group AND
//   - Device is managed AND
//   - Risk level is low

user.isMemberOf("Engineering") 
&& device.enrolled 
&& device.managed 
&& riskLevel == "LOW"

// Step-up for high-risk
if (riskLevel == "HIGH" || request.ip.isSuspicious) {
  stepUp("TOTP")  // Require additional TOTP
}
```

### OIDC Integration (any app)

```javascript
// Node.js app with Okta OIDC
const { ExpressOIDC } = require("@okta/oidc-middleware");

const oidc = new ExpressOIDC({
  issuer: "https://YOUR_ORG.okta.com/oauth2/default",
  client_id: process.env.OKTA_CLIENT_ID,
  client_secret: process.env.OKTA_CLIENT_SECRET,
  redirect_uri: "https://app.example.com/callback",
  scope: "openid profile email groups",
});

app.use(oidc.router);

// Protected route
app.get("/admin", oidc.ensureAuthenticated(), (req, res) => {
  const user = req.userContext.userinfo;
  const groups = user.groups || [];
  
  if (!groups.includes("Admins")) {
    return res.status(403).json({ error: "Not in Admins group" });
  }
  
  res.json({ message: `Hello, admin ${user.email}!` });
});
```

---

## Pomerium (Open-Source IAP)

**What it is**: Open-source Identity-Aware Proxy — self-hosted alternative to Cloudflare Access for protecting internal apps.

```yaml
# pomerium/config.yaml
authenticate_service_url: https://authenticate.example.com

idp_provider: google  # or okta, azure, github, etc.
idp_client_id: YOUR_GOOGLE_CLIENT_ID
idp_client_secret: YOUR_GOOGLE_CLIENT_SECRET

# Define what to protect and who can access it
policy:
  - from: https://app.example.com
    to: http://internal-app:3000
    policy:
      - allow:
          and:
            - email:
                is: alice@example.com
            - email:
                is: bob@example.com
  
  - from: https://admin.example.com
    to: http://admin-panel:8080
    policy:
      - allow:
          and:
            - groups:
                has: "admins@example.com"  # Google Group
```

---

## Open Policy Agent (OPA): The Policy Engine

**What it is**: A general-purpose policy engine used by many Zero Trust systems for authorization decisions.

```rego
# policies/access.rego
package access

import future.keywords.if
import future.keywords.in

default allow = false

# Main allow rule
allow if {
    user_is_authenticated
    user_in_group
    device_is_compliant
    resource_allowed
    not time_restricted
}

user_is_authenticated if {
    input.user.authenticated == true
    input.user.mfa_verified == true
}

user_in_group if {
    required_group := data.resource_groups[input.resource.name]
    required_group in input.user.groups
}

device_is_compliant if {
    input.device.managed == true
    input.device.encrypted == true
    input.device.os_update_days < 30
}

resource_allowed if {
    allowed_methods := data.resource_methods[input.resource.name]
    input.request.method in allowed_methods
}

# Block access outside business hours for sensitive resources
time_restricted if {
    input.resource.sensitivity == "high"
    hour := time.clock(time.now_ns())[0]
    hour < 8
}

time_restricted if {
    input.resource.sensitivity == "high"
    hour := time.clock(time.now_ns())[0]
    hour >= 20
}
```

```bash
# Query OPA
echo '{
  "input": {
    "user": {"authenticated": true, "mfa_verified": true, "groups": ["engineering"]},
    "device": {"managed": true, "encrypted": true, "os_update_days": 5},
    "resource": {"name": "prod-api", "sensitivity": "medium"},
    "request": {"method": "GET"}
  }
}' | curl -s -X POST http://opa:8181/v1/data/access -d @-

# Response: {"result": {"allow": true}}
```

---

## Choosing the Right Tool Combination

```
Small startup (< 50 users):
  Identity: Google Workspace (built-in)
  ZTNA: Cloudflare Zero Trust (free for 50 users)
  Device: Basic MDM via Cloudflare WARP
  Cost: ~$0

Growing startup (50-200 users):
  Identity: Okta (Starter plan)
  ZTNA: Cloudflare Zero Trust Teams
  Device: Jamf Now (Mac) + Intune (Windows)
  Service mesh: Consul Connect (open source)
  Cost: ~$5-15/user/month

Enterprise (200+ users):
  Identity: Okta Enterprise / Microsoft Entra ID Premium P2
  ZTNA: Zscaler Private Access / Cloudflare for Teams
  Device: Jamf Pro + CrowdStrike Falcon
  Network: Illumio microsegmentation
  PAM: CyberArk
  SIEM: Splunk / Microsoft Sentinel
  Service mesh: Istio on Kubernetes
  Cost: $25-50+/user/month
```
