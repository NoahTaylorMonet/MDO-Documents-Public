# Defender for Office 365 — False Positive / False Negative Submissions in GCC High

## Summary

In GCC High, the FedRAMP High / IL5 data boundary changes how false-positive (FP) and false-negative (FN) submissions to Microsoft work in Defender for Office 365 (MDO):

- End users **cannot** send reported messages to Microsoft from Outlook — that routing option is disabled by policy. Reported mail can only be delivered to a designated **internal reporting mailbox**.
- User reporting in **Microsoft Teams** is **not available**.
- Admins **can** open the Submissions page at `https://security.microsoft.us`, but Microsoft only runs **email authentication and policy checks** on the submission. **No payload detonation and no human grader analysis** are performed, because tenant content is not permitted to leave the sovereign cloud boundary.
- Every admin submission returns the same result text:
  > *"Further investigation needed. Your tenant doesn't allow data to leave the environment, so nothing was found during the initial scan. You'll need to contact Microsoft support to have this item reviewed."*

To obtain a true Microsoft verdict on an FP/FN — or to push intelligence back into the global service — a Microsoft Support case must be opened from the GCC High admin portal and the sample provided through that channel.

## What works vs. what doesn't in GCC High

| Capability | GCC High |
|---|---|
| User "Report" button routed to Microsoft | **Disabled — boundary** |
| User "Report" button routed to internal reporting mailbox | Yes (only option) |
| User reporting in Microsoft Teams | **No** |
| Admin Submissions page UI at `security.microsoft.us` | Yes |
| Submission runs SPF/DKIM/DMARC + policy hit checks | Yes |
| Submission runs URL / attachment detonation | **No** |
| Submission runs human grader (SME) review | **No** |
| Submission returns an actionable Microsoft verdict | **No — "contact support"** |
| Tenant Allow/Block List entry from submission | Yes (GA July 2024) |
| Automated Investigation & Response (AIR) on user reports | Yes (Defender for Office 365 Plan 2) |
| Defender unified RBAC for MDO | **Not GA** |
| Security Copilot Phishing Triage Agent | **No** |

## Recommended FP/FN workflow in GCC High

1. **Have end users report** with the built-in Outlook **Report** button (Phishing / Junk / Not Junk). Messages land in your designated internal reporting mailbox and appear in the Defender portal under **Actions & submissions → Submissions → User reported**.
2. **Triage internally.** SecOps reviews on the **User reported** tab, uses the **Email entity page** and Threat Explorer (P2) to validate verdict, and clicks **Mark as and notify** to send the user a templated result.
3. **Take immediate action with Tenant Allow/Block List (TABL).** From the Submissions flow or directly under **Email & collaboration → Policies & rules → Threat policies → Tenant Allow/Block Lists**, create an allow entry (for confirmed FP) or block entry (for confirmed FN) on the URL, sender, domain, or file hash. This is enforced inside the boundary and does not require Microsoft analysis.
4. **For anything that needs a true Microsoft verdict** — a recurring FP that TABL can't fully solve, a missed malware payload that needs to be globally remediated, or a campaign you want investigated — **open a Microsoft Support case** using the steps in the next section.
5. **Optional:** Also file the submission from the Defender portal (**Submissions → Submit to Microsoft for analysis**). It will return *"Further investigation needed,"* but it creates an artifact, applies your TABL entry, and gives you a Submission ID to reference in the support case.

## Step-by-step: Opening a Microsoft Support case from GCC High

> Support cases for GCC High are opened from the **GCC High Microsoft 365 admin center**, not commercial admin.microsoft.com.

### Prerequisites (one-time)

- The admin opening the case must hold **Global Administrator**, **Service Administrator**, or **Helpdesk Administrator** in the GCC High tenant.
- The admin account must have a **valid, reachable email address** that Microsoft Support can call back on. If the sign-in UPN is on a `.mil` or non-routable domain, add an **Alternative email address** on the user record (Admin center → Users → Active users → *user* → Manage username and email / Alternative email address) **before** opening the case.
- **GCC High customer support is not inside the service accreditation boundary** and does not carry FedRAMP / DoD SRG / ITAR / IRS 1075 / CJIS data-handling assurances. Do **not** attach controlled, sensitive, or classified content to a support case until the assigned agent's authorization to view it is confirmed. Where possible, share only headers, hashes, URLs, and sanitized samples.

### Open the case

1. Sign in to the **GCC High admin center** at **https://portal.office365.us** with a Global / Service / Helpdesk Admin account.
2. In the left nav, expand **Support** and select **New service request** (or click the **Help & support** "?" icon, then **Contact support**).
3. **Product / service:** select **Microsoft Defender for Office 365** (or **Exchange Online Protection**, depending on the issue).
4. **Problem type:** choose **Anti-spam, anti-malware, or anti-phish** → **False positive** *or* **False negative — message was not detected**.
5. **Title:** clear and specific, e.g., *"GCCH MDO false negative — phishing message delivered to inbox, NMID `<guid>`, need payload review."*
6. **Description:** include the items in the checklist below.
7. **Attachments:** attach the message as `.eml` or `.msg` (in Outlook: **File → Save As → Outlook Message Format - Unicode**). If the file is sensitive, hold it back and tell the agent it will be provided through an approved secure channel after caller validation.
8. **Severity:** **A (Critical)** only for active in-progress attacks affecting many users; otherwise **B (High)** for production phishing/malware issues, **C (Non-critical)** for tuning requests.
9. **Confirm callback details** (phone + email) and submit. A service request number (`REQ` / `TrackingID`) is returned — record it.

### What to include in the case (checklist)

For each message, attach or paste:

- The original `.eml` or `.msg` file (saved from Outlook with full headers).
- **Network Message ID** — from header `X-MS-Exchange-Organization-Network-Message-Id` (or `X-MS-Office365-Filtering-Correlation-Id` for quarantined items).
- **Internet Message ID**, **Date/Time received (UTC)**, **Sender address**, **Recipient address(es)**, **Subject**.
- Current Microsoft verdict (e.g., "Delivered to Inbox," "Quarantined as Phish," "Delivered to Junk") and the expected verdict.
- The Defender portal **Submission ID** if it was also filed through Submissions.
- For URLs: full URL(s) and any redirect chain observed.
- For attachments: SHA-256 hash of each file (`Get-FileHash -Algorithm SHA256 <file>` in PowerShell), plus the file itself if sharing is approved.
- Any **TABL entries** already created in response.
- A short statement of **business impact** (number of users affected, role, mission criticality).

### After the case is open

- The case is routed to **Microsoft 365 for US Government support**. They validate caller identity against the email/phone on file.
- For payload analysis specifically, the support engineer coordinates with the **Microsoft Defender research team** on the commercial side to perform detonation/grading on the sample provided *through the support channel* (not from tenant data).
- Track the case in **Admin center → Support → View service requests**.
- Once a verdict is returned, Microsoft updates global protection signals where applicable. Mirror the result in the tenant by adjusting the **TABL** entry or anti-spam/anti-phish policy as needed.

## Interim mitigations available entirely inside the boundary

Most FPs and FNs can be fully mitigated with built-in MDO controls without a Microsoft round-trip:

- **Tenant Allow/Block List**: allow or block by sender, domain, URL, or file hash (10,000 block / 5,000 allow entries per category in MDO P2).
- **Anti-phishing policy → Trusted senders and domains**: stops impersonation flagging for recurring legitimate senders.
- **Anti-spam policy → Allowed/Blocked senders and domains**: per-policy overrides.
- **Mail flow (transport) rules**: targeted bypass or block with conditions (e.g., header value, sender IP).
- **AIR (Defender for Office 365 P2)**: automatically investigates user-reported phish and recommends remediation that admins approve from the Action Center — runs entirely inside the GCCH boundary.
- **Threat Explorer (P2)**: hunt for related messages by NMID, URL, sender, or attachment hash and remediate in bulk (soft-delete, hard-delete, move to junk) — also fully in-boundary.

## Reference URLs

- Defender portal (GCC High): **https://security.microsoft.us**
- Admin center (GCC High): **https://portal.office365.us**
- Docs — admin Submissions page: https://learn.microsoft.com/defender-office-365/submissions-admin
- Docs — user reported settings (note the GCCH restriction): https://learn.microsoft.com/defender-office-365/submissions-user-reported-messages-custom-mailbox
- Docs — submission result definitions ("Further investigation needed"): https://learn.microsoft.com/defender-office-365/submissions-result-definitions
- Docs — MDO for US Government: https://learn.microsoft.com/defender-office-365/mdo-gov
- Docs — Opening support cases for GCC High and DoD: https://learn.microsoft.com/troubleshoot/microsoft-365/admin/miscellaneous/support-cases-for-gcc-high-dod
