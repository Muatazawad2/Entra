# Client-Owned Devices — Protecting Microsoft 365 Data

How to let staff work at client sites on **client-owned, client-managed laptops** while keeping your organization's Microsoft 365 data off those devices — two designs compared, validated in a lab tenant, with step-by-step deployment.

## Contents

- [Client-Owned-Device-Access-Playbook.md](Client-Owned-Device-Access-Playbook.md) — the full guide: the problem, both routes, lab results, step-by-step deployment and lessons learned.
- `Images/` — diagrams and redacted lab screenshots:
  - `route2-cloud-pc-diagram.png`, `route1-browser-mdca-diagram.png` — the two routes at a glance.
  - `laptop-blocked-error-53000.png` — an unmanaged laptop blocked from Outlook on the web.
  - `cloud-pc-redirection-defaults.png` — clipboard, drive and printer redirection already off inside a Cloud PC.
  - `mdca-monitored-notice.png`, `mdca-work-profile-prompt.png`, `mdca-download-blocked-log.png` — what the browser route looks like to a worker, and a blocked download.

## What is the problem?

Contoso staff are placed at client sites and use laptops that the **client** owns and manages. They need Contoso's Outlook, Teams, OneDrive and SharePoint, but Contoso must keep its data off devices it does not manage and keep personal devices out.

The obvious fixes don't work:

- **Enroll the laptop in Contoso's tenant** — a Windows device can be managed by only one organization, and the client already manages it.
- **Intune app protection (MAM) for Windows** — requires a device that isn't enrolled in any organization.
- **Browser session control alone (Defender for Cloud Apps)** — without a Contoso Edge work profile, sessions fall back to a reverse proxy (`*.mcas.ms`) with a "monitored" notice, which workers find confusing.

## The two routes

| | Route 1: Browser + Defender for Cloud Apps | Route 2: Windows 365 Cloud PC |
|---|---|---|
| How it works | Workers use the client laptop's browser; Defender for Cloud Apps blocks downloads | Workers use a Contoso-managed Cloud PC; the laptop is only a screen and keyboard |
| Where Contoso data lives | Displayed on the client laptop | Stays in the Cloud PC |
| Blocks unmanaged and personal devices | Yes | Yes |
| Stops data reaching the laptop | Partly — downloads blocked, content still displayed | Yes — clipboard, file transfer and printing blocked |
| Worker experience | Profile prompt, monitoring notice, proxy address | A normal Windows desktop |
| Extra licensing | Defender for Cloud Apps | Windows 365 per worker |

**Recommendation:** lead with **Route 2**. Use Route 1 — or the lighter SharePoint/Exchange web-only access — only for workers who can't be given a Cloud PC. Keep the two routes in separate user groups.

## Quick start — Route 2

1. Create a **break-glass** account and exclude it from every Conditional Access policy.
2. Create a worker group and assign **Windows 365 Enterprise** licences.
3. Create a **provisioning policy**: Microsoft Entra join, Microsoft-hosted network, single sign-on on.
4. In Intune (settings catalog), set **Do not allow Clipboard / drive / client printer redirection** = Enabled.
5. Create the **Conditional Access** rule: All resources **except** Windows 365, Azure Virtual Desktop and Windows Cloud Login → require a compliant or hybrid-joined device → **Report-only**.
6. Validate in the sign-in logs, pilot with a dedicated test user, then switch the rule **On**.

Full step-by-step guidance for both routes is in [Client-Owned-Device-Access-Playbook.md](Client-Owned-Device-Access-Playbook.md).

## Notes

- Validated in a lab tenant. Test in your own environment — in report-only and with a dedicated test user — before enforcing.
- Windows 365 disables clipboard, drive, USB and printer redirection by default; the Intune policy makes that explicit and reportable.
- Someone can still photograph the screen. Windows 365 screen capture protection and watermarking reduce, but don't remove, that risk.

---

**Developer**: Dr Muataz Awad
