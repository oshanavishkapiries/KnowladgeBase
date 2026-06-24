# 07 — Edge Cases & Failure Modes in Zero Trust

## The Paradox of Zero Trust

Zero Trust is designed to reduce security risk — but a **poorly implemented Zero Trust system** can create new failure modes:
- Over-restrictive policies block legitimate work
- Auth failures lock users out of critical systems during incidents
- Single points of failure in the policy engine
- Security theater without actual improvement

Understanding failure modes is essential before and during implementation.

---

## Failure Mode 1: Policy Engine Outage

The Policy Engine (Cloudflare Access, Okta, etc.) is now a **critical dependency** for all resource access.

```
SCENARIO: Cloudflare Access is down for 15 minutes

Traditional network:
  Impact: Zero — users connect directly through VPN

Zero Trust (poorly designed):
  Impact: ALL internal apps unreachable for 15 minutes
  Users: Cannot access anything
  Business: $$$$ in lost productivity

Zero Trust (well designed):
  Impact: Minimal — break-glass procedures activated
  Duration: 15 minutes of manual verification
```

### Mitigation Strategies

```
1. Choose a provider with high SLA (Cloudflare: 99.99% uptime)

2. Design break-glass procedures:
   - Emergency admin bypass (requires physical token + manager approval)
   - Direct network access for critical teams (documented, audited)
   - Secondary policy engine with auto-failover

3. Architect for resilience:
   - Cache auth decisions at PEP (e.g., for 5 minutes)
   - Use multiple IdP providers (primary: Okta, fallback: AD)
   - Regional failover for auth infrastructure

4. Test failover procedures regularly:
   - Quarterly fire drill: "What if Okta is down for 2 hours?"
   - Document and test every break-glass procedure
```

---

## Failure Mode 2: MFA Lockout

Users can be locked out of their accounts due to:
- Lost phone (authenticator app gone)
- SIM swap / phone number change
- Time sync issues with TOTP
- MFA fatigue attacks (so many denials → user gives up)

```
SCENARIO: CISO's phone is stolen at 2 AM during a security incident
  They need urgent access to the SIEM
  Their Okta is locked (MFA device lost)
  
Poor Zero Trust: No emergency access → incident response delayed by hours
Good Zero Trust: 
  1. Recovery codes (printed and in sealed envelope in safe)
  2. Manager-approved emergency access (documented process)
  3. Backup MFA method (hardware key in office)
```

### Mitigation: Account Recovery Procedures

```
Every user should have:
  - Recovery codes (8-10 single-use backup codes, printed)
  - Secondary MFA device registered (e.g., both phone + hardware key)
  - Manager can initiate temporary bypass (time-limited, requires approval)

Ops procedures:
  1. Help desk cannot reset MFA without identity verification
     → Video call + employee ID + manager confirmation
  2. MFA reset creates immediate security alert
  3. All recovery actions logged and audited
  4. User notified by multiple channels when MFA is changed
```

---

## Failure Mode 3: Device Compliance Deadlock

A user's device fails compliance (e.g., OS patch required) and now they:
- Can't access their email to read the IT notice telling them to update
- Can't access the MDM portal to approve the update
- Can't get help desk support (it's behind Zero Trust too)

```
SCENARIO: Bob's MacBook needs a security patch.
  Company policy: OS must be within 7 days of latest patch.
  Bob's Mac: 10 days behind.
  Result: Bob is locked out.
  
  Bob tries to:
  - Email IT: BLOCKED (email is behind Zero Trust)
  - Access Jira: BLOCKED (Jira is behind Zero Trust)
  - Go to IT portal: BLOCKED (portal is behind Zero Trust)
  - Call IT: ✅ Phone works
```

### Mitigation: Compliance Grace Period and Escape Hatch

```
Policy design:
  - Grace period: 7 days warning before enforcement (not immediate lockout)
  - Compliance check failures: warning mode first (log, alert, but don't block)
  - Exception: Allow access to MDM self-service portal even on non-compliant devices
  - Exception: Allow access to IT help portal even on non-compliant devices
  - Exception: Allow email for IT communications even on non-compliant devices

Implementation:
  # Cloudflare Access example
  # Policy for "General Apps": require compliant device
  # Policy for "IT Help Portal": allow regardless of device compliance
  
  Policy: IT Help Portal
    Action: Allow
    Include: All authenticated employees
    # NO device compliance requirement on this specific app
```

---

## Failure Mode 4: Lateral Movement Within Zero Trust

Zero Trust reduces lateral movement but doesn't eliminate it if:
- Microsegmentation is incomplete (some east-west traffic is unrestricted)
- A privileged service account is compromised
- mTLS is not enforced (services trust each other implicitly)

```
ATTACK SCENARIO: Zero Trust Lateral Movement

Step 1: Attacker phishes credentials for marketing employee
Step 2: Attacker passes MFA (using stolen push notification approval)
Step 3: Attacker accesses internal marketing wiki (allowed by policy)
Step 4: Attacker discovers API key for shared service in wiki page
Step 5: Attacker uses API key to access orders service
Step 6: Orders service trusts all requests from internal API key
Step 7: Attacker exfiltrates order data

Root Cause: The API key in the wiki wasn't Zero Trust — it was a long-lived 
            secret with implicit trust.
```

### Mitigation

```
1. Zero standing privileges: No long-lived API keys stored anywhere
   → Use SPIFFE/Vault dynamic secrets instead

2. Service-to-service auth, not just user auth:
   → mTLS between all services (Istio/Linkerd)
   → SPIFFE workload identity

3. Classify all secrets — any secret in a wiki is a secret policy violation
   → DLP scanning on wikis/code for secrets
   → Tools: truffleHog, GitGuardian, GitHub secret scanning

4. Just-in-time network access for services:
   → API gateway with per-request auth (not long-lived keys)
```

---

## Failure Mode 5: Token Theft & Session Hijacking

Zero Trust issues short-lived tokens, but tokens can still be stolen if:
- XSS vulnerability in a web app extracts the JWT
- Token is logged in access logs (never log auth tokens!)
- Phishing site proxies the session (Evilginx/Muraena attack)

```
ADVANCED PHISHING ATTACK (Adversary-in-the-Middle):

Traditional phishing: Steal password
AITM phishing: Steal session token AFTER authentication

1. User visits evil-example.com (looks like real Okta)
2. Evil site proxies REAL Okta authentication
3. User completes MFA successfully (against real Okta)
4. Evil site captures the session token/cookie
5. Attacker reuses token to impersonate user

This bypasses: passwords, TOTP, push MFA
Does NOT bypass: FIDO2/WebAuthn (origin-bound cryptography)
```

### Mitigation

```
1. Use FIDO2/WebAuthn for high-value applications — AITM-proof by design

2. Implement token binding:
   - Bind session token to device certificate
   - Token is invalid if used from different device

3. Continuous session validation:
   - Re-validate device posture every 15-30 minutes
   - Short token lifetimes (15-60 minutes)
   - Detect impossible travel (login from London then NYC in 10 min)

4. Certificate-based session binding (Cloudflare Access):
   - Short-lived service tokens
   - Device certificate required alongside session token
```

---

## Failure Mode 6: Insider Threat Within Zero Trust

Zero Trust limits what insiders can access, but a legitimate privileged user can still cause damage.

```
INSIDER THREAT SCENARIO:

Departing engineer with prod DB access:
  - On last day, copies prod database backup to personal cloud storage
  - Zero Trust granted them this access legitimately
  - Policy didn't restrict WHERE data could be sent
  
Zero Trust alone didn't prevent this.
```

### Mitigation

```
1. Data Loss Prevention (DLP):
   - Inspect outbound traffic for sensitive data patterns
   - Block uploads of large data to personal cloud (Google Drive, Dropbox)
   - Alert on unusual data exfiltration volumes

2. Continuous monitoring for anomalous behavior:
   - Baseline: Engineer normally downloads 10MB/day from prod DB
   - Alert: Engineer downloads 2GB on final day
   
3. Departing employee procedures:
   - Revoke access before announcing departure (if threat suspected)
   - Accelerated deprovisioning workflow
   - Offboarding checklist: revoke all ZTNA access, revoke device certs

4. Session recording for privileged access:
   - All production database sessions recorded
   - Privileged SSH sessions recorded and searchable
   - Tools: Teleport, CyberArk, BeyondTrust
```

---

## Failure Mode 7: Zero Trust Configuration Drift

Over time, Zero Trust policies accumulate exceptions, legacy rules, and stale entries:

```
Year 1:  Clean policy — 50 users, 20 applications
Year 2:  + contractor exceptions + legacy app bypass + 3 new teams
Year 3:  + acquired company users + emergency access rules that weren't removed
Year 4:  + nobody knows what half the policies do + auditors are concerned
```

### Mitigation: Policy as Code

```yaml
# Store all Zero Trust policies in version control
# File: policies/cloudflare-access/prod-api.yaml

metadata:
  name: "Production API Access"
  owner: "platform-team@example.com"
  reviewed_date: "2026-01-15"
  review_interval_days: 90
  ticket: "SEC-1234"
  
policy:
  action: allow
  include:
    - email_domain: example.com
    - group: "engineering"
  require:
    - mfa: true
    - device_managed: true
  exclude:
    - email: "contractor123@vendor.com"  # Exception expires 2026-07-01 (SEC-5678)
```

```python
# Policy audit script — find stale policies
import yaml
import glob
from datetime import datetime, timedelta

for policy_file in glob.glob("policies/**/*.yaml", recursive=True):
    with open(policy_file) as f:
        policy = yaml.safe_load(f)
    
    reviewed = datetime.strptime(policy["metadata"]["reviewed_date"], "%Y-%m-%d")
    interval = policy["metadata"].get("review_interval_days", 90)
    
    if (datetime.now() - reviewed).days > interval:
        print(f"STALE POLICY: {policy_file} — last reviewed {reviewed.date()}")
```

---

## Failure Mode 8: Supply Chain Attack via Zero Trust Tools

Your Zero Trust tools are themselves attack vectors. If Okta, Cloudflare, or your IdP is compromised, attackers can bypass everything.

```
REAL-WORLD EXAMPLE: Okta Breach (2022)
  - Lapsus$ group breached Okta support system
  - Gained access to some customer tenant admin panels
  - Okta customers' access policies could potentially be modified

LESSON: Zero Trust vendors are not immune to compromise.
```

### Mitigation

```
1. Defense in depth — don't rely on Zero Trust alone:
   - Even if ZTNA is bypassed, protect resources at the application level
   - App-level authentication (not just network-level ZTNA)
   - Database-level access controls

2. Monitor for anomalous policy changes:
   - Alert on any Zero Trust policy change
   - Require approval for production policy changes
   - Store policy change history (immutable audit log)

3. Vendor security assessment:
   - Review vendor's SOC 2 Type II report annually
   - Check vendor's incident response history
   - Understand data sharing in vendor contracts

4. Multi-vendor approach for highest-sensitivity resources:
   - Two different ZTNA products for different resource tiers
   - Attacker must compromise both to reach crown jewels
```

---

## Operational Runbook: Zero Trust Incident Response

```
INCIDENT: Users report they cannot access [Application]

Step 1 (0-5 minutes): Triage
  □ Is it one user or all users?
  □ Is it one app or all apps?
  □ When did it start?
  □ What changed recently (policy, deployment, etc.)?

Step 2 (5-15 minutes): Isolate
  □ Check ZTNA vendor status page (status.cloudflare.com, etc.)
  □ Check IdP status (status.okta.com)
  □ Check recent policy changes in audit log
  □ Check device compliance status for affected users

Step 3 (15-30 minutes): Restore access
  □ If vendor outage: activate break-glass access
  □ If policy misconfiguration: revert to last known good policy
  □ If device compliance issue: grant temporary exception (approved)

Step 4 (Post-incident): Analysis
  □ Root cause analysis
  □ Update break-glass procedures if needed
  □ Add monitoring to catch this issue earlier next time
  □ Document in incident tracking
```

---

## Monitoring Alerts: What to Alert On

```
Critical (PagerDuty wake-up):
  - Identity Provider (Okta/AD) is unreachable
  - Policy Engine is returning errors > 1%
  - Certificate expired on auth infrastructure
  - Admin account activity outside business hours

High (On-call notification):
  - MFA bypass detected (authentication without expected MFA type)
  - Impossible travel (login from two distant cities < 2 hours apart)
  - Bulk access policy changes by non-admin user
  - > 10 policy denials for same user in 5 minutes

Medium (Next-business-day):
  - Device non-compliant for > 7 days
  - Service account with expired credentials
  - Policy not reviewed in > 90 days
  - Unused access grant not cleaned up in 30 days

Low (Weekly report):
  - Users without backup MFA method
  - Devices missing EDR agent
  - Applications not yet behind ZTNA
  - Stale guest/contractor accounts
```

---

## Literature Final Thought

> *Designing Data-Intensive Applications* (Kleppmann): "Hardware faults, software errors, and human error" are the three main causes of unreliable systems. Zero Trust implementations fail for exactly these three reasons:
>
> - **Hardware/infrastructure**: IdP outage, network partition
> - **Software errors**: Policy misconfiguration, tool bugs
> - **Human error**: Overly permissive exceptions, forgotten cleanup
>
> The solution is the same as for data systems: **redundancy, automation, and systematic testing of failure modes** — before those failures happen in production.
