# Device Code Flow — Phishing Defense

Guidance, KQL, and a staged playbook to discover and safely block **OAuth 2.0 Device Code Flow** phishing in Microsoft Entra ID — without breaking legitimate workflows.

## Contents

- [Device-Code-Flow-Playbook.md](Device-Code-Flow-Playbook.md) — the full staged playbook (Discover → Classify → Simulate → Exclude → Enforce → Monitor & Automate).
- `Images/` — supporting infographics:
  - `1.png`, `2.png` — discovering device code sign-ins in the Entra portal (two filter methods).
  - `3.png` — legitimate flow vs. phishing attack, and why MFA alone doesn't stop it.
  - `4.png` — the six-stage defense playbook at a glance.

## What is the risk?

Device Code Flow is designed for input-constrained devices (smart TVs, IoT, CLI tools). Attackers abuse it: they generate a device code, send the victim a legitimate-looking Microsoft prompt, and the victim unknowingly authorizes the *attacker's* session — completing real MFA in the process. Because the victim authenticates against the real Microsoft URL, "just add MFA" does not stop it. The effective control is **Conditional Access → Authentication Flows**.

## Quick start — discover device code sign-ins (KQL)

Run in Microsoft Sentinel or Log Analytics:

```kusto
union isfuzzy=true SigninLogs, AADNonInteractiveUserSignInLogs
| where TimeGenerated >= ago(30d)
| where tostring(AuthenticationProtocol) =~ 'deviceCode'
| summarize DeviceCodeSignInCount=count(), FirstSeen=min(TimeGenerated), LastSeen=max(TimeGenerated)
```

For full per-event detail (user, app, IP, country, risk), see the **Discover** section of the playbook.

## How to block device code phishing in Entra

1. **Discover** every device code sign-in (KQL above or Entra sign-in logs).
2. **Classify** legitimate vs. legacy/unknown usage.
3. **Simulate** — create a Conditional Access policy: Authentication Flows = Device Code Flow → Block, in **Report-only**.
4. **Exclude** — add a minimal, documented exception group for genuine business needs.
5. **Enforce** — switch the policy from Report-only to On.
6. **Monitor & Automate** — alert on new or suspicious device code usage and recertify exclusions.

Full step-by-step guidance is in [Device-Code-Flow-Playbook.md](Device-Code-Flow-Playbook.md).

## Notes

- This is guidance + detection content; it complements, and does not replace, native Entra protections.
- Tune the KQL thresholds and country/risk filters to your tenant to reduce false positives.

---

**Developer**: Dr Muataz Awad
