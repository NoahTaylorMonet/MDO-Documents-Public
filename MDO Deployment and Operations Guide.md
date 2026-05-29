# Microsoft Defender for Office 365: MDO Deployment and Operations Guide

A practical reference for deploying and operating Microsoft Defender for Office 365 — covering policy configuration (presets, impersonation, outbound spam), Unified RBAC, mail-flow and email authentication foundations, TABL and submissions workflows, alert investigation and AIR, quarantine and alert tuning, Sentinel and Graph API automation, advanced hunting, troubleshooting, Attack Simulation Training, and content filtering.

---

<details open>
<summary><h2>0. Current State and Recommendations</h2></summary>

> 🧭 **Source:** Derived from review of the *MBS New Architecture Diagram* (Visio) describing the existing Boeing mail security architecture across the two GCC High tenants.

### Current State

The existing Boeing Mail Boundary Service (MBS) architecture, as represented in the current Visio, is structured as a mirrored inbound/outbound design across two separate Microsoft 365 GCC High tenants and a perimeter chain of on-premises mail security infrastructure.

**Tenants and zones**

- **Two GCC High tenants** operate side by side:
  - `boeing.com` (primary user tenant)
  - `boeing.onmicrosoft.com` (secondary tenant)
- Each tenant is depicted as its own trust boundary, with **independent** policy, quarantine, and management surfaces.
- Three concentric trust zones surround the tenants:
  1. **Internet / M365 cloud edge**
  2. **Dedicated Security Zone** — contains a stack of Windows servers per side that function as the third-party Secure Email Gateway (SEG) and management hosts. This is the first hop for inbound mail and the last hop for outbound mail.
  3. **Boeing Perimeter** — edge SMTP/connector tier between the security zone and the internal network.
  4. **BEN (Boeing Enterprise Network)** — internal users (Outlook clients) and Linux-based application/service mail submitters.

**Mail flow**

- **Inbound:** Internet → Dedicated Security Zone (SEG) → Boeing Perimeter → GCC High tenant → mailbox.
- **Outbound:** Mailbox → GCC High tenant → Boeing Perimeter → Dedicated Security Zone (SEG) → Internet.
- All flows in the diagram are **strict one-way arrows** — inbound and outbound paths are physically and logically segmented, with no bidirectional channels in the same path.
- Linux service hosts inside BEN submit application mail through the same egress chain as user mail.

**Filtering and security posture (current)**

- Primary spam, phishing, malware, URL, and attachment filtering happens on the **third-party SEG** in the Dedicated Security Zone — not in EOP/MDO.
- EOP, by virtue of MX terminating downstream of the SEG, sees the SEG's IP as the sending IP for nearly all inbound mail. SPF/DMARC/IP-reputation evaluation in EOP is therefore degraded today unless Enhanced Filtering for Connectors is configured.
- Quarantine, end-user release, and admin investigation workflows live primarily on the SEG appliances, not in MDO.
- The diagram contains no representation of MDO, Defender XDR, Sentinel, the Report Message experience, or Conditional Access — these protections are either not deployed, or not part of the current as-built reference.

**Implications**

- Two tenants = two independent MDO/EOP control planes that must be kept in sync.
- The SEG tier is the dominant control point today; MDO is currently a downstream consumer with reduced signal fidelity.
- SOC tooling (logging, hunting, response) is anchored to the SEG, not to Microsoft 365 Defender / Sentinel.

---

### Recommendations: Move to MDO

The recommendation is a phased transition that establishes MDO as the authoritative mail security control plane in both tenants, with the third-party SEG either removed or relegated to a coexistence role.

**1. Establish MDO as the primary filter (per tenant)**

- Apply **Standard preset** to all users and **Strict preset** to executives, privileged accounts, and high-value targets in *both* `boeing.com` and `boeing.onmicrosoft.com`.
- Enable **Safe Links**, **Safe Attachments** (Dynamic Delivery), **anti-phish impersonation protection**, **Mailbox Intelligence**, and **Spoof Intelligence** in each tenant.
- Stand up a **config-as-code** baseline (PowerShell / Microsoft Graph) so policy state stays identical across the two tenants.

**2. Decide the SEG disposition**

| Option | Outcome | Recommendation |
|--------|---------|----------------|
| **Full MX cutover to EOP/MDO** | MDO is first-touch; full signal fidelity, full feature efficacy | Preferred end state |
| **SEG remains in front (coexistence)** | Lower disruption, but requires Enhanced Filtering for Connectors (skip-listing) so MDO sees the true sender IP | Acceptable as a transitional state only |
| **SEG behind EOP** | Not recommended — duplicates filtering and complicates quarantine | Avoid |

If coexistence is required during transition, **Enhanced Filtering for Connectors must be enabled** so SPF, DMARC, and IP-reputation signals reach MDO correctly.

**3. Email authentication hardening (prerequisite to enforcement)**

- Confirm or publish **SPF, DKIM, and DMARC** for every sending domain and subdomain — including domains used by the BEN Linux service senders.
- Drive DMARC from `p=none` → `p=quarantine` → `p=reject`, with reporting (RUA/RUF) ownership assigned.

**4. Outbound and service-account mail flow**

- Inventory the BEN Linux senders and create **dedicated outbound connectors** (certificate-based preferred) with the appropriate authentication.
- Validate they will not trip the High Risk Delivery Pool once MDO outbound spam policies are in force.
- Replace any wide IP allow-lists left over from the SEG era — these are a common source of MDO efficacy loss.

**5. Operations, quarantine, and end-user experience**

- Move quarantine review and release into **MDO quarantine policies** with appropriate end-user permissions and notification (digest) cadence.
- Deploy the **Report Message / Report Phishing** add-in (or built-in Outlook reporting) and route user reports to MDO Submissions for AIR processing.
- Update helpdesk runbooks to reference MDO/Defender XDR rather than the SEG console.

**6. SOC integration**

- Connect both tenants to **Microsoft Sentinel (GCC High)** via the Microsoft 365 Defender connector.
- Re-author existing SEG-based detections against the MDO tables (`EmailEvents`, `EmailUrlInfo`, `EmailAttachmentInfo`, `UrlClickEvents`).
- Use **Threat Explorer**, **AIR**, and **Advanced Hunting** as the primary investigation surfaces.

**7. Cross-tenant considerations**

- Mail between `boeing.com` and `boeing.onmicrosoft.com` is treated as external by default. Configure cross-tenant connectors with Enhanced Filtering, and keep Safe Links / Safe Attachments scanning **on** for inter-tenant mail.

**8. Licensing and GCC High validation**

- Confirm **MDO Plan 2** (or G5 Security) coverage for every user, shared, and resource mailbox in both tenants, including service accounts with mailboxes.
- Validate GCC High feature parity for any planned capability (e.g., Attack Simulation Training, certain Defender XDR automation features may lag commercial).
- Ensure firewall and proxy allow-lists use the **GCC High endpoints** (`*.office365.us`, `*.protection.office365.us`), and communicate the GCC High Safe Links rewrite domain to end users.

**9. Target-state architecture**

The target architecture replaces the Dedicated Security Zone Windows stack with **EOP + MDO inside each tenant boundary**, adds Defender XDR + Sentinel as a shared SOC plane consuming both tenants, and shows explicit MX, outbound smart host, Enhanced Filtering, and user-reporting flows. The current Visio should be revised to reflect this end state and to mark the SEG components for decommission.

---

</details>

---

<details open>
<summary><h2>1. Initial MDO Policy Configuration</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Email & collaboration → Policies & rules → **Threat policies**

### Recommended Starting Point: Preset Security Policies

Microsoft provides two managed preset policy profiles that cover the core MDO protection surface with Microsoft-recommended settings. These should be the starting point before any custom policy configuration.

| Profile | Protection Level | Recommended Use |
|---------|-----------------|-----------------|
| **Standard protection** | Balanced | Most users in production |
| **Strict protection** | Aggressive | High-value targets, executives, privileged accounts |

**Where:** Threat policies → **Preset security policies**

Both profiles configure the following simultaneously:
- Anti-phishing policy (impersonation, spoof intelligence, mailbox intelligence)
- Anti-spam policy (inbound and outbound)
- Anti-malware policy
- Safe Links policy
- Safe Attachments policy

> ⚠️ **Important:** Preset policies are managed by Microsoft. Settings within preset profiles cannot be individually customized. When custom policies are required, those users or groups must be excluded from preset policies and added to custom.

---

### Policy Application Order

MDO applies policies in priority order. Preset security policies are evaluated before custom policies, and the first matching policy wins.

```
Strict preset (if assigned) → Standard preset (if assigned) → Custom policies (by priority) → Default policy
```

- Assign Strict to high-value targets first.
- Use custom policies to handle edge cases or exceptions.
- The built-in default policy applies to anyone not covered by a higher-priority policy.

---

### Recommended Pre-Deployment Configuration Checklist

- [ ] Assign Standard protection to all users
- [ ] Assign Strict protection to executives, privileged accounts, and high-value targets
- [ ] Validate MX records point to Exchange Online, not a third-party gateway
- [ ] Confirm SPF, DKIM, and DMARC are configured for all sending domains
- [ ] Validate DMARC rollout plan (p=none -> p=quarantine -> p=reject) with reporting ownership
- [ ] Confirm the Defender for Office 365 data connector in Microsoft Sentinel
- [ ] Confirm audit logging is enabled — Microsoft Purview portal → **Audit** → verify the banner reads *Auditing is on*; if not, select **Start recording user and admin activity**. Unified audit log ingestion is required for MDO investigation, Threat Explorer admin-action tracking, and Sentinel enrichment.
- [ ] Define quarantine ownership model (who can review/release, and escalation path)

---

### Priority Accounts and Strict Assignment Criteria

Assign **Strict protection** only where operationally justified:

- Executives and executive support staff
- Privileged administrators
- Users with repeated targeting history

Be prepared for **increased quarantine** volume for Strict-assigned users due to those settings.

---

### Configuration Analyzer Validation Step

After policy assignment, run Configuration Analyzer to identify controls below Standard/Strict recommended posture, then remediate or document approved exceptions.

---

### Secure by Default effect on mail delivery

Secure by Default ensures that messages identified as **malware** or **high-confidence phishing** are always quarantined, regardless of allow-list overrides (transport rules, safe sender lists, user allows). This behavior **cannot be disabled.**

- Transport rules with SCL -1 do **not** bypass Secure by Default verdicts.
- User-level safe senders do **not** bypass Secure by Default verdicts.
- Only TABL allow entries for the exact indicator (submitted via Submissions) can release future matches — and only if Microsoft agrees with the assessment.

---

### Protection Scope Beyond Email (SPO, OneDrive, Teams)

Safe Attachments for SharePoint/OneDrive/Teams and Safe Links for supported Office clients are separate global toggles and are commonly missed during deployment. Make sure to verify that these are enabled using the instructions below.

**Safe Attachments for SPO/ODB/Teams:**
- Navigate: `security.microsoft.com` → Threat policies → **Safe Attachments** → *Global settings*.
- Enable **Turn on Defender for Office 365 for SharePoint, OneDrive, and Microsoft Teams**.
- Enable **Turn on Safe Documents for Office clients** (E5 required).

**Safe Links scope:**
- Confirm Safe Links policies include **Teams** and **Office 365 apps** toggles.
- Validate click-time protection via Advanced Hunting `UrlClickEvents` with non-email click sources.

---

### Outbound Spam Policy and Auto-Forwarding Control

Outbound mail controls are important to contain compromised accounts and prevent data exfiltration via auto-forwarding.

**Where:** Threat policies → **Anti-spam** → *Anti-spam outbound policy*.

Key settings:

| Setting | Recommended Value |
|---|---|
| External message limit (per hour) | Default 500 (tune to business pattern) |
| Internal message limit (per hour) | Default 1,000 |
| Daily message limit | Default 1,000 |
| Action when user exceeds limit | **Restrict the user from sending email** |
| Automatic forwarding | **Off — forwarding disabled** (unless business-justified) |
| Notify admin on restricted user | Enabled, to SOC distribution |

Restricted users appear in Defender portal → **Review** → *Restricted users*. Include checking this queue in the daily operations checklist.

---

### Impersonation Protection Configuration

Preset policies enable impersonation protection framework, but protected users and domains must still be populated manually.

**Where:** Threat policies → **Anti-phishing** → preset or custom policy → *Impersonation*.

Configure:

1. **Protected users** — add up to 350 high-value internal mailboxes (executives, VIPs, high level admins, etc). Populate display name + SMTP address.
2. **Protected domains** — add all internal sending domains plus any external partners you frequently work with.
3. **Mailbox intelligence** — ensure *Enable intelligence for impersonation protection* is on.
4. **Trusted senders and domains** — use sparingly to reduce false positives for known legitimate lookalikes. We want to avoid whitelisting too many domains/senders in general.
5. **Action on impersonation detection** — Quarantine (preset default).

Review the protected user list quarterly as org structure changes.

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-office-365/preset-security-policies
https://learn.microsoft.com/en-us/defender-office-365/anti-phishing-policies-about
https://learn.microsoft.com/en-us/defender-office-365/safe-links-about
https://learn.microsoft.com/en-us/defender-office-365/safe-attachments-about
https://learn.microsoft.com/en-us/defender-office-365/secure-by-default
https://learn.microsoft.com/en-us/defender-office-365/mdo-for-spo-odb-and-teams-configure
https://learn.microsoft.com/en-us/defender-office-365/outbound-spam-policies-configure
https://learn.microsoft.com/en-us/defender-office-365/anti-phishing-mdo-impersonation-insight
```

</details>

---

<details>
<summary><h2>1a. Permissions and Access (Unified RBAC)</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → **Permissions** → *Microsoft Defender XDR* → **Roles**

### Configuring Unified RBAC (URBAC) Permissions in MDO

Microsoft Defender Unified RBAC (URBAC) is the preferred permissions model for MDO. It replaces the legacy Email & collaboration role groups and protection-related Exchange Online roles with a single, more granular set of permissions to follow least privilege best practices.

> ⚠️ **IMPORTANT:** Once URBAC is activated for the *Defender for Office 365* workload, the legacy Email & collaboration roles page at `security.microsoft.com/emailandcollabpermissions` is no longer used for enforcement. Configure (or import) all URBAC roles **before** activation. Entra ID Security Administrator / Global Administrator continue to work post-activation.

**Where:** `security.microsoft.com` → **Permissions** → *Microsoft Defender XDR* → **Roles** → **Create custom role** (or **Import roles** to migrate legacy role groups).

#### URBAC building blocks

1. **Permission groups** — broad categories:
   - *Security operations* (day-to-day SOC work: alerts, incidents, email data, quarantine, advanced actions)
   - *Security posture* (Secure Score, vulnerability management)
   - *Authorization and settings* (manage roles, security settings, system settings)
2. **Permissions** — individual read / manage toggles within each group.
3. **Assignments** — bind a role to **Entra security groups / users** scoped to specific **data sources** (e.g., Defender for Office 365 only, or all workloads).

#### MDO URBAC permissions Explained

| Permission (path) | Level | What it grants |
|---|---|---|
| Security operations \ Security data \ Security data basics | Read | Incidents, alerts, hunting, reports, submissions |
| Security operations \ Security data \ Alerts | Manage | Triage, assign, classify, start investigations |
| Security operations \ Security data \ Response | Manage | Approve/dismiss pending actions; manage auto allow/block lists |
| Security operations \ Raw data (Email & collaboration) \ Email & collaboration metadata | Read | Email headers, Threat Explorer data, advanced hunting email schema |
| Security operations \ Raw data (Email & collaboration) \ Email & collaboration content | Read | Preview and download email bodies and attachments |
| Security operations \ Security data \ Email & collaboration quarantine | Manage | View and release quarantined mail |
| Security operations \ Security data \ Email & collaboration advanced actions | Manage | Soft/hard delete, move to junk or inbox (remediation) |
| Authorization and settings \ Security settings \ Core security settings | Read / Manage | View or edit MDO policies (anti-phish/spam/malware, Safe Links/Attachments) |
| Authorization and settings \ Security settings \ Detection tuning | Manage | Manage TABL and alert tuning rules |
| Authorization and settings \ System settings | Read / Manage | Portal system configuration |
| Authorization and settings \ Authorization | Read / Manage | Create/modify URBAC roles |

---

### Recommended Tiered URBAC Role Model for MDO

Below are some examples of custom URBAC roles for various personas. Assign via Entra security groups, not individual users.

#### Tier 1 — SOC Analyst (MDO triage)

**Persona:** Front-line SOC analyst triaging MDO incidents, classifying alerts, releasing end-user reports back to Microsoft. No configuration or mailbox remediation rights.

| Permission | Level |
|---|---|
| Security operations \ Security data \ Security data basics | Read |
| Security operations \ Security data \ Alerts | Manage |
| Security operations \ Raw data (Email & collaboration) \ Email & collaboration metadata | Read |
| Authorization and settings \ Security settings \ Core security settings | Read |
| Authorization and settings \ System settings | Read |

#### Tier 2 — SOC Investigator (email content + remediation)

**Persona:** Senior SOC analyst performing email investigations, previewing messages, executing soft-delete/hard-delete via Threat Explorer. Cannot change MDO policies.

Inherits all Tier 1 permissions, plus:

| Permission | Level |
|---|---|
| Security operations \ Raw data (Email & collaboration) \ Email & collaboration content | Read |
| Security operations \ Security data \ Email & collaboration quarantine | Manage |
| Security operations \ Security data \ Email & collaboration advanced actions | Manage |
| Security operations \ Security data \ Response | Manage |

#### Tier 3 — MDO Engineer / Platform Owner (policy & tuning)

**Persona:** Platform/engineering owner responsible for MDO policy configuration, TABL management, and alert-tuning-rule lifecycle. Full operational authority for the MDO workload without tenant-wide admin rights.

Inherits all Tier 2 permissions, plus:

| Permission | Level |
|---|---|
| Authorization and settings \ Security settings \ Core security settings | Manage |
| Authorization and settings \ Security settings \ Detection tuning | Manage |
| Authorization and settings \ System settings | Read and manage |

#### Read-Only Auditor / Compliance

**Persona:** Auditor, compliance, or reporting role; must see policy configuration, alerts, and reports without any change rights.

| Permission | Level |
|---|---|
| Security operations \ Security data \ Security data basics | Read |
| Security operations \ Raw data (Email & collaboration) \ Email & collaboration metadata | Read |
| Authorization and settings \ Security settings \ Core security settings | Read |
| Authorization and settings \ System settings | Read |

#### URBAC Administrator (role management only)

**Persona:** Identity / RBAC administrator who manages URBAC role definitions and assignments but does not perform MDO operations.

| Permission | Level |
|---|---|
| Authorization and settings \ Authorization | Read and manage |

---

### URBAC Deployment for MDO

1. **Inventory existing role assignments** — export current Email & collaboration role group members (Defender portal → Permissions → Email & collaboration roles → Export) and Exchange Online protection-related role group members (EAC → Roles).
2. **Design role catalog** — Make sure the tiered roles match your SOC operating model.
3. **Create Entra security groups** — one per URBAC role (e.g., `SG-MDO-SOC-Tier1`, `SG-MDO-Engineer`). Nest users via group membership only.
4. **Create the custom roles** — for each tier, create the role, select permissions per the tables above, and add an **Assignment** that maps the role to its Entra group and the **Defender for Office 365** data source.
5. **Remove legacy assignments** — after activation is stable, remove users from the legacy Email & collaboration role groups.

---

### Best practices for URBAC

- **Prefer Entra security groups** for all assignments; never assign individual users directly. We would also want to make sure PIM is configured for these groups for no standing access.
- **Least privilege first.** Do not assign *Security Administrator* or *Global Administrator* as a shortcut to MDO access — both are tenant-wide and far broader than any tier role above.
- **Separation of duties.** The URBAC Administrator role should be distinct from the MDO Engineer role. The same person should not be able to both grant permissions and configure policies in production.
- **IMPORTANT NOTE: Powershell** Exchange Online PowerShell and Security & Compliance PowerShell continue to enforce **Exchange Online roles** — they are **not** governed by URBAC. Any automation scripts must still map to Exchange Online role groups.
- **Review cadence.** Review role membership quarterly.

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-xdr/manage-rbac
https://learn.microsoft.com/en-us/defender-xdr/create-custom-rbac-roles
https://learn.microsoft.com/en-us/defender-xdr/custom-permissions-details
https://learn.microsoft.com/en-us/defender-xdr/compare-rbac-roles
https://learn.microsoft.com/en-us/defender-xdr/activate-defender-rbac
```

</details>

---

<details>
<summary><h2>1b. Mail Flow and Authentication Foundations</h2></summary>

> 🧭 **Navigation:** Defender portal → **Email authentication settings** · `security.microsoft.com` → Email & collaboration → Policies & rules → **Enhanced filtering** · Microsoft 365 admin center → Settings → **Org settings**

### Email Authentication Rollout

Ensure for each active sending domain and subdomain:

1. SPF configured and validated against all legitimate senders
2. DKIM enabled and signing confirmed for production traffic
3. DMARC deployed in phases (`p=none` -> `p=quarantine` -> `p=reject`) with monitoring at each step

---

### DKIM Enablement Steps

DKIM must be configured per custom sending domain. The high-level rollout in the pre-deployment checklist expands to:

1. **Publish DKIM CNAME records** — Defender portal → Email authentication settings → **DKIM** → select domain → copy the two CNAME records (`selector1._domainkey`, `selector2._domainkey`) and publish in public DNS.
2. **Wait for DNS propagation**, then return to the DKIM page and toggle **Sign messages for this domain with DKIM signatures** to *Enabled*.
3. **Validate** signing with a message trace or by inspecting `Authentication-Results` header in a sent message (should show `dkim=pass`).
4. **Repeat for every custom sending domain**, including parked/unused domains (publish a null DKIM record to prevent abuse).

---

### Enhanced Filtering for Connectors (SEG Coexistence)

If for any reason we must keep the existing mail system temporarily, Enhanced Filtering must be configured. Without it, spoof intelligence, sender reputation, and authentication checks are degraded.

**Where:** `security.microsoft.com` → Email & collaboration → Policies & rules → **Enhanced filtering**.

**Steps:**
1. Identify the inbound connector receiving SEG traffic.
2. Enable **Automatically detect and skip the last IP address**, or specify the SEG's public IP ranges.
3. Apply to all users, or a scoped recipient group during pilot.
4. Validate via message trace — the `Authentication-Results` and received headers should reflect the original sender IP.

---

### Priority Account Protection (Tenant Setting)

Priority account protection is a **distinct tenant setting** from Strict preset assignment. It flags mailboxes for differentiated alerting, reporting, and elevated ML models.

**Where:** Microsoft 365 admin center → Settings → **Org settings** → *Priority account protection*.

- Enable the feature.
- Tag up to 250 mailboxes as priority accounts (typically executives, finance, legal, or high level admins).
- Combine with Strict preset assignment for layered protection.
- Validate visibility in Threat Explorer's *Priority accounts* view.

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-office-365/email-authentication-about
https://learn.microsoft.com/en-us/defender-office-365/email-authentication-dkim-configure
https://learn.microsoft.com/en-us/defender-office-365/mdo-about-connectors
https://learn.microsoft.com/en-us/microsoft-365/admin/setup/priority-accounts
https://learn.microsoft.com/en-us/purview/audit-log-enable-disable
```

</details>

---

<details>
<summary><h2>2. Tenant Allow/Block List (TABL) Operations</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Email & collaboration → Policies & rules → Threat policies → Rules section → **Tenant Allow/Block Lists**

### What TABL Is For

TABL allows administrators to manually override MDO filtering verdicts for specific indicators. It is the primary customer-managed enforcement surface for high-confidence, active threats.

TABL is not a bulk indicator repository. It is a precision enforcement cache for indicators that require immediate action ahead of or beyond what MDO's automatic filtering provides.

---

### TABL Tabs and Capacities (MDO Plan 2)

| Tab | MDO Plan 2 Limit | Notes |
|-----|-----------------|-------|
| Domains & addresses | 15,000 (5,000 allow / 10,000 block) | Covers From address (P2 sender) |
| URLs | 15,000 (5,000 allow / 10,000 block) | Max 250 characters per entry |
| Files | 15,000 (5,000 allow / 10,000 block) | SHA256 hash only |
| IP addresses | 15,000 (5,000 allow / 10,000 block) | **IPv6 only** — IPv4 not yet supported |
| Spoofed senders | 1,024 combined allow + block | Never expire |

> 📌 **IPv4 Note:** IPv4 entries are not supported in TABL today. Timing for IPv4 support is on the Microsoft roadmap and subject to change — confirm current status in the Message Center and the TABL documentation before planning around it. Use the **connection filter policy** for IPv4 IP allow/block controls in the interim.

---

### Entry Behavior by Type

| Entry Type | Block Behavior | Allow Behavior | Expiry |
|-----------|---------------|---------------|--------|
| Domains & addresses | Marked high confidence phishing, quarantined | Overrides spam/phish verdicts | Configurable, default 30 days |
| URLs | Marked high confidence phishing, quarantined | Overrides URL filter | Configurable, default 30 days |
| Files | Marked as malware, quarantined | Overrides file filter | Configurable, default 30 days |
| IP addresses (IPv6) | Rejected at service edge | Bypasses IP filtering | Configurable, default never expire |
| Spoofed senders | Marked as phishing | Overrides spoof intelligence | Never expire |

---

### When to Use TABL vs. Let MDO Decide

| Situation | Action |
|-----------|--------|
| Indicator is active, confirmed malicious, high confidence | Promote into TABL block |
| Indicator is dormant, unconfirmed | Leave in Sentinel; let MDO full stack evaluate |
| Known-good sender is being blocked (false positive) | Submit via Submissions page first; TABL allow as needed |
| Malware or high confidence phishing false positive | Must use Submissions page — cannot add allow directly in TABL |

---

### TABL Operational Guidance

- Always submit false positives via the **Submissions page** before adding allow entries manually. Microsoft's analysis may result in automatic resolution.
- Avoid adding allow entries for malware or high confidence phishing directly — the portal will not permit it; use Submissions.
- Block entries for domains and addresses also prevent internal users from **sending** to those targets.
- Monitor TABL capacity by indicator type as part of routine operations.

---

### Non-TABL Enforcement Controls

| Control | Limit | Supports IPv4 | Use Case |
|---------|-------|--------------|----------|
| Connection Filter — IP Allow List | 1,273 entries | ✅ Yes | Trusted sending infrastructure |
| Connection Filter — IP Block List | 1,273 entries | ✅ Yes | Block by sending IP |
| Anti-Spam — Allowed sender list | 1,024 per policy | ✅ Yes | Per-policy sender allow |
| Anti-Spam — Blocked sender list | 1,024 per policy | ✅ Yes | Per-policy sender block |
| Exchange Transport Rules | 300 rules per tenant | ✅ Yes | Advanced conditional mail flow logic |

> 💡 **Note:** Connection filter IP Allow List bypasses spam filtering but messages are still scanned for malware and high confidence phishing.

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-about
https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-email-spoof-configure
https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-urls-configure
https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-files-configure
https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-ip-addresses-configure
https://learn.microsoft.com/en-us/defender-office-365/connection-filter-policies-configure
```

</details>

---

<details>
<summary><h2>3. Alerts and Investigation</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Incidents & alerts → **Incidents** → Filter by Service source: Defender for Office 365

### How MDO Alerts Flow

```
MDO detects malicious email activity
    ↓
Alert generated (phishing, malware, campaign, etc.)
    ↓
Defender XDR correlates into Incidents
    ↓
Incident appears with email evidence and entities
    ↓
Telemetry ingested into Microsoft Sentinel
    ↓
Sentinel correlates with indicator inventory and XDR signals
    ↓
Automation promotes indicator to TABL if warranted
```

---

### Investigating an MDO Incident

**Step 1: Filter to MDO Alerts**

| Do | Notes |
|----|-------|
| Go to Incidents & alerts → Incidents | |
| Add filter → Service source → Defender for Office 365 | Isolates email-sourced incidents |
| Review the incident list | Look for campaigns, repeated entities, or high-severity items |

**Step 2: Open an Incident**

| Do | Notes |
|----|-------|
| Click on an incident | |
| Review the Attack story tab | Shows timeline and how entities relate |
| Review the Evidence tab | Shows affected emails, URLs, files, senders |

**Step 3: Investigate the Email**

| Do | Notes |
|----|-------|
| Click an email entity | Opens email summary |
| Review sender, subject, delivery action, and delivery location | Was it delivered, quarantined, or blocked? |
| Check the Detection technology field | Shows what triggered the verdict: Safe Links, Safe Attachments, spoof intelligence, etc. |
| Open Threat Explorer for deeper analysis | Full message trace and delivery details |

**Email Investigation Checklist:**
- [ ] Was the message delivered or quarantined?
- [ ] What detection triggered the verdict?
- [ ] Are other messages in the same campaign?
- [ ] Has the sender or URL appeared in other incidents?
- [ ] Is the indicator already in TABL or Sentinel?
- [ ] Do any recipients need remediation?

**Step 4: Make a Decision**

| Finding | Action |
|---------|--------|
| Active threat, confirmed malicious | Soft-delete from mailboxes, promote indicator into TABL |
| False positive (legitimate mail blocked) | Submit via Submissions page, add allow entry to TABL if needed |
| Suspicious but unclear | Continue investigating; do not close; flag for Sentinel correlation |
| Campaign identified | Review campaign view in Threat Explorer; consider bulk remediation |

---

### Automated Investigation and Response (AIR)

AIR automatically investigates certain MDO alert types and proposes or executes remediation actions.

**Where:** `security.microsoft.com` → **Action center** (Pending / History tabs) and the **Investigations** tab on an incident.

Initiated AIR playbooks:

| Playbook | Trigger |
|---|---|
| User-reported phish | User reports a message as phishing |
| URL click verdict | Safe Links detects a malicious click post-delivery |
| Malware ZAP | ZAP remediates malware post-delivery |
| Phish ZAP | ZAP remediates phish post-delivery |
| Weaponized URL in email | MDO detects a weaponized URL in a delivered message |
| User compromise | Unusual sending behavior indicates potential compromise |

**Approval model:**

- AIR investigations complete automatically; remediation actions default to **pending approval** and require a reviewer to approve in Action center. 
- Action center queue should be reviewed at least daily during normal operations.
- For high-confidence actions (e.g., soft-delete of ZAP'd phish), consider enabling **automatic approval** via configuration — document risk acceptance first.

---

### Threat Explorer

> 🧭 **Navigation:** `security.microsoft.com` → Email & collaboration → **Explorer**

Threat Explorer provides deep message-level analysis and is the primary tool for email investigation.

**Useful Explorer Views:**

| View | What It Shows |
|------|--------------|
| All email | Full view of inbound mail with filtering |
| Phish | Emails detected as phishing |
| Malware | Emails with malware detections |
| Campaigns | Grouped attack campaigns with shared IoCs |

**Common Investigation Filters:**

| Filter | Use Case |
|--------|----------|
| Sender domain | Find all mail from a specific domain |
| Delivery action | Blocked, delivered, quarantined, junked |
| Detection technology | What rule or signal triggered the verdict |
| URL | Find emails containing a specific URL |
| File hash | Find emails containing a matched attachment |

---

### Tuning Mindset

After reviewing incidents, ask the following:

| Pattern | Signal |
|---------|--------|
| Same sender keeps generating false positive alerts | Add precise allow entry or adjust policy |
| Same indicator keeps appearing in new incidents | Consider promoting to TABL block |
| Detection fired but mail was actually clean | Submit as false positive, evaluate exclusion |
| Campaign detected with shared URL or attachment | Block the URL or file hash in TABL |
| Alert resolved quickly by automation | Validate automation is working as expected |

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-office-365/threat-explorer-about
https://learn.microsoft.com/en-us/defender-office-365/mdo-sec-ops-guide
https://learn.microsoft.com/en-us/defender-office-365/remediate-malicious-email-delivered-office-365
```

</details>

---

<details>
<summary><h2>4. Submissions and False Positive Management</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Email & collaboration → **Submissions**

### When to Use the Submissions Page

The Submissions page is the preferred method for reporting false positives and false negatives to Microsoft. It is also **required** for certain allow entry types that cannot be created directly in TABL.

| Scenario | Use Submissions |
|----------|----------------|
| Good email was blocked as malware | ✅ Required — cannot add allow for malware directly in TABL |
| Good email was blocked as high confidence phishing | ✅ Required |
| Malicious email was delivered (false negative) | ✅ Recommended |
| You want Microsoft to improve their filtering | ✅ Yes |
| Active threat you want blocked immediately | Use TABL block directly; also submit for analysis |

---

### Submission Types

| Tab | What You Submit | Result |
|-----|----------------|--------|
| Emails | An email message ID or .eml file | Microsoft reviews and may create allow entry |
| Email attachments | File hash or attachment | Used for file false positives |
| URLs | A specific URL | Microsoft reviews; allow entry created if clean |

---

### Submission Workflow

1. Navigate to the Submissions page
2. Select the appropriate tab (Emails, Email attachments, URLs)
3. Submit the item with appropriate classification (I've confirmed it's clean / I've confirmed it's a threat)
4. If clean: select **Allow this message / URL / file** on confirmation
5. Allow entry is added to TABL automatically if Microsoft agrees with the assessment
6. Monitor submission status under the Submitted items tab

---

### User-Reported Messages Configuration

Configure the end-user reporting experience so submitted messages land in the SOC queue and are forwarded to Microsoft for analysis. **NOTE: Government tenants will NOT have the option to send to Microsoft**

**Where:** `security.microsoft.com` → Email & collaboration → **Submissions** → *User reported settings* tab.

**Configuration steps:**

1. **Reporting destination** — select one of:
   - *Microsoft and my reporting mailbox* (recommended — Microsoft analyzes AND SOC sees the report)
   - *Microsoft only*
   - *My reporting mailbox only*
2. **Reporting mailbox** — specify the monitored mailbox (e.g., `phishreport@contoso.com`). Ensure it is a shared mailbox the SOC queue tool ingests.
3. **Reporting tool** — choose:
   - *Use the built-in Report button in Outlook* (recommended for modern Outlook; no add-in required)
   - *Microsoft Report Message add-in* or *Report Phishing add-in* (for classic clients)
   - *A third-party reporting tool* — specify header requirements
4. **User experience** — customize before/after reporting pop-ups with organization-specific messaging.
5. **Validate** — have a pilot user report a test message; confirm it appears in Submissions → *User reported* tab and in the reporting mailbox.

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-office-365/submissions-admin
https://learn.microsoft.com/en-us/defender-office-365/submissions-outlook-report-messages
https://learn.microsoft.com/en-us/defender-office-365/submissions-user-reported-messages-custom-mailbox
```

</details>

---

<details>
<summary><h2>5. Exclusions and Policy Overrides</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Email & collaboration → Policies & rules → **Threat policies**

### When to Use Exclusions

Exclusions reduce alert noise for known-good senders, domains, or files. Use them when:

- A sender is consistently generating false positives from an authorized source
- A file type or hash is being blocked that is known to be safe
- A delivery scenario is disrupting a legitimate mail flow

⚠️ **Warning:** Over-exclusion creates blind spots. Prefer targeted, scoped exclusions over broad ones. Review exclusions periodically.

---

### Exclusion Types by Control

| Control | Exclusion Method |
|---------|-----------------|
| Anti-phishing | Trusted senders and domains (per policy) |
| Anti-spam | Allowed sender list / allowed domain list (per policy) |
| Safe Attachments | Policy exceptions by user, group, or domain |
| Safe Links | Do-not-rewrite list (per policy) |
| TABL | Allow entries (globally applied to all policies) |
| Connection Filter | IP Allow List (bypasses spam filtering for those IPs) |

---

### Exclusion Guidance

- **TABL allow entries** are the broadest — they override filtering for that entity across all policies. Use sparingly for confirmed-clean items.
- **Per-policy exclusions** (anti-phishing trusted senders, anti-spam allowed senders) are scoped and less risky. Prefer these for ongoing operational exceptions.
- **Safe Links do-not-rewrite list** entries prevent URLs from being wrapped by Safe Links but do not allow malicious URLs — this is for legitimate internal or partner URLs that break when rewritten.
- Document every exclusion with a business justification, owner, and review date.

---

### Periodic Exclusion Review Checklist

- [ ] Review TABL allow entries — are they still needed? Still valid?
- [ ] Review anti-spam allowed sender and domain lists — any stale entries?
- [ ] Review Safe Links do-not-rewrite lists — are all URLs still legitimate?
- [ ] Review anti-phishing trusted sender lists — any domains that should no longer be trusted?
- [ ] Confirm connection filter IP Allow List entries are still valid sending infrastructure

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-office-365/create-safe-sender-lists-in-office-365
https://learn.microsoft.com/en-us/defender-office-365/safe-links-about#do-not-rewrite-the-following-urls-lists-in-safe-links-policies
https://learn.microsoft.com/en-us/defender-office-365/anti-phishing-policies-about
```

</details>

---

<details>
<summary><h2>6. Sentinel Integration and Automation</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Settings → **Microsoft Sentinel** (connector)
> Also: `portal.azure.com` → Microsoft Sentinel → Data connectors

### Connecting MDO to Microsoft Sentinel

Defender XDR connects to Sentinel through a single unified data connector. Refer to Sentinel and MDO Indicator Deployment Guide.

---

### MDO Tables Available in Sentinel (via Streaming API)

| Table | What It Contains |
|-------|-----------------|
| EmailEvents | All inbound/outbound email activity — sender, recipient, subject, delivery action, detection technology |
| EmailAttachmentInfo | Attachment metadata and file hash information per message |
| EmailUrlInfo | URLs found in email messages |
| EmailPostDeliveryEvents | Post-delivery actions — ZAP, manual remediation, quarantine actions |

---

### Key MDO Sentinel Automation Capabilities

The Sentinel integration enables the following automated response actions as part of the mail architecture:

| Automation Trigger | Action |
|-------------------|--------|
| Indicator detected in active email | Promote indicator to TABL block via Logic App / playbook |
| High-confidence phishing campaign detected | Soft-delete delivered emails via Graph API |
| Dormant indicator re-observed | Alert analyst, evaluate for re-promotion |
| TABL capacity threshold approached | Alert operations team |
| TABL entry past dormancy window | Remove from TABL; retain in Sentinel |


### Key Graph API capabilities for MDO automation:

| Action | Endpoint |
|--------|----------|
| Soft delete email from mailbox (move to Deleted Items) | `POST /v1.0/users/{id}/messages/{messageId}/move` with `destinationId: deleteditems` |
| Hard delete email from mailbox (Purges folder, recoverable only via retention/hold) | `POST /v1.0/users/{id}/messages/{messageId}/permanentDelete` (v1.0; not available in US Gov L4/L5 or China 21Vianet clouds) |
| Trigger email incident remediation | Via Defender XDR API actions |
| Add entry to TABL | Via Exchange Online PowerShell from playbook |
| Remove entry from TABL | Via Exchange Online PowerShell from playbook |
| Run Advanced Hunting query | `POST /v1.0/security/runHuntingQuery` |

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/azure/sentinel/microsoft-365-defender-sentinel-integration
https://learn.microsoft.com/en-us/defender-xdr/streaming-api
https://learn.microsoft.com/en-us/graph/api/resources/security-api-overview
https://learn.microsoft.com/en-us/defender-office-365/remediate-malicious-email-delivered-office-365
```

</details>

---

<details>
<summary><h2>7. Operations and Health Monitoring</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Reports → **Email & collaboration reports**
> Also: Microsoft 365 admin center → Health → **Service health**

### What to Monitor

| Area | Where to Check | What to Watch For |
|------|---------------|-----------------|
| Policy health | Threat policies page | Policies with no assignments or conflicting priorities |
| TABL capacity | Tenant Allow/Block Lists | Percentage of capacity used per indicator type |
| Quarantine volume | Review → Quarantine | Spikes in quarantined mail volume |
| Detection trends | Email & collaboration reports | Changes in phishing, malware, or spam detection rates |
| ZAP activity | Email & collaboration reports → ZAP report | Messages remediated post-delivery |
| Safe Links / Safe Attachments | Reports → Defender for Office 365 | Detonation activity and blocked URLs |
| Service health | Microsoft 365 admin center | MDO service availability and incidents |

---

### TABL Capacity Monitoring

TABL capacity should be tracked per indicator type. Engineering should alert before the following thresholds:

| Indicator Type | Capacity | Suggested Alert Threshold |
|---------------|---------|--------------------------|
| Domains & addresses | 15,000 | 80% (12,000 entries) |
| URLs | 15,000 | 80% (12,000 entries) |
| Files | 15,000 | 80% (12,000 entries) |
| IP addresses (IPv6) | 15,000 | 80% (12,000 entries) |
| Spoofed senders | 1,024 | 75% (768 entries) |

**Consider setting separate thresholds for Allow Entries and Block Entries due to different entry limits.**
---

### Routine Operations Checklist

**Daily/Weekly:**
- [ ] Review new incidents from MDO in Sentinel and Defender XDR
- [ ] Review automation health — check for failed playbooks or stuck promotions
- [ ] Check for TABL capacity warnings
- [ ] Review any quarantine releases or false positive submissions

**Monthly:**
- [ ] Review TABL allow entries — remove any that are no longer needed
- [ ] Review exclusions across policies for stale or overly broad entries
- [ ] Review detection trend reports for anomalies
- [ ] Review automation logs for patterns in failed or deferred promotions
- [ ] Confirm dormancy expiry logic is removing stale TABL entries as expected

---

### Quarantine Policies and End-User Experience

Quarantine policies control what end users can see and do with quarantined messages, and how admins interact with them. These are separate from the anti-spam/anti-phish policies that route mail to quarantine.

**Where:** `security.microsoft.com` → Email & collaboration → Policies & rules → Threat policies → **Quarantine policies**.

**Built-in quarantine policies:**

| Policy | Permissions |
|---|---|
| `AdminOnlyAccessPolicy` | Admin-only; users see nothing |
| `DefaultFullAccessPolicy` | Users can view, preview, and request release |
| `DefaultFullAccessWithNotificationPolicy` | Full access + quarantine notifications emailed to user |

**Recommended mapping to verdicts:**

| Verdict | Recommended Policy |
|---|---|
| High confidence phishing | `AdminOnlyAccessPolicy` |
| Malware | `AdminOnlyAccessPolicy` |
| Phishing | `DefaultFullAccessPolicy` (request release) |
| High confidence spam | `DefaultFullAccessPolicy` |
| Spam | `DefaultFullAccessWithNotificationPolicy` |
| Bulk | `DefaultFullAccessWithNotificationPolicy` |

**End-user notifications (Quarantine digest) IMPORTANT: Determine if you want to notify end users or not with quarantine notifications. Consider workload on admins managing quarantine:**

- Notifications are controlled by the **quarantine policy** assigned to each verdict. Only policies built on `DefaultFullAccessWithNotificationPolicy` (or custom policies with notifications enabled) send digests.
- Configure digest frequency via **Quarantine notifications** global settings (or per policy) — supported values are **every 4 hours**, **daily**, or **weekly**. Daily is a reasonable default for production.
- Brand notifications with org name/logo and SOC contact.

**Admin operations:**

- Bulk release and preview require the Quarantine Administrator or Security Administrator role.
- For release actions on *admin-only* policies, enable **approve release** workflow so requested releases route to the SOC queue.
- Monitor release-rate trends — a sudden spike often indicates a false-positive pattern worth tuning at policy level.

---

### Defender XDR Alert Tuning for MDO

Use Defender XDR alert tuning rules — not policy exclusions or TABL allows — whenever a recurring MDO alert pattern is a confirmed benign-true-positive.

**Where:** `security.microsoft.com` → Settings → Microsoft Defender XDR → **Rules → Alert tuning**, or via *Tune alert* on any MDO alert page.

**Decision order for recurring MDO alert noise:**

1. **Is the alert caused by misconfiguration or mail-flow issue?** → Fix root cause (connector, EFC, DMARC, DKIM). No tuning.
2. **Is the alert driven by an authorized internal automation or vendor?** → Create an alert tuning rule scoped to the evidence (sender, URL domain, attachment hash, recipient group).
3. **Is the allow truly tenant-wide for a specific indicator?** → Use TABL allow (via Submissions page) instead of alert tuning.
4. **Do entire user groups need reduced visibility rather than reduced alerts?** → Use Unified RBAC scoping, not suppression.

**Guardrails:**

- Avoid broadly suppressing: *Email messages containing malicious URL*, *Malware campaign*, *User compromised*, *Suspicious email forwarding activity*, *Suspicious email sending patterns*.
- Record rule name, detector, scope, justification, owner, and review date for every tuning rule created.
- Review all tuning rules monthly — delete zero-match rules.

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-office-365/reports-email-security
https://learn.microsoft.com/en-us/defender-office-365/quarantine-admin-manage-messages-files
https://learn.microsoft.com/en-us/defender-office-365/zero-hour-auto-purge
https://learn.microsoft.com/en-us/defender-xdr/investigate-alerts#tune-an-alert
```

</details>

---

<details>
<summary><h2>8. Advanced Hunting</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Hunting → **Advanced hunting**

### Key MDO Tables

| Table | What It Captures |
|-------|-----------------|
| `EmailEvents` | All email activity: sender, recipient, subject, delivery action, detection technology, network message ID |
| `EmailAttachmentInfo` | Attachments per message: file name, SHA256 hash, malware verdict |
| `EmailUrlInfo` | URLs per message: URL value, click action, threat type |
| `EmailPostDeliveryEvents` | Post-delivery changes: ZAP, manual soft/hard delete, quarantine move |
| `UrlClickEvents` | Safe Links click data: user, URL, action taken |

---

### Sample Queries

**All phishing emails delivered in the last 7 days:**
```kql
EmailEvents
| where Timestamp > ago(7d)
| where ThreatTypes has "Phish"
| where DeliveryAction == "Delivered"
| project Timestamp, SenderFromAddress, RecipientEmailAddress, Subject, ThreatTypes, DetectionMethods, NetworkMessageId
| order by Timestamp desc
```

**Emails containing a specific indicator (URL or domain) in the last 30 days:**
```kql
EmailUrlInfo
| where Timestamp > ago(30d)
| where Url contains "suspiciousdomain.com"
| join kind=inner EmailEvents on NetworkMessageId
| project Timestamp, SenderFromAddress, RecipientEmailAddress, Subject, Url, DeliveryAction
```

**Emails with a specific file hash in attachments:**

> 💡 `EmailAttachmentInfo.SHA256` is often not populated — prefer `SHA1`, or match on either.

```kql
let targetSha1  = "replace_with_actual_sha1_hash";
let targetSha256 = "replace_with_actual_sha256_hash";
EmailAttachmentInfo
| where Timestamp > ago(30d)
| where SHA1 == targetSha1 or SHA256 == targetSha256
| join kind=inner EmailEvents on NetworkMessageId
| project Timestamp, SenderFromAddress, RecipientEmailAddress, Subject, FileName, SHA1, SHA256, DeliveryAction
```

**Messages ZAP'd in the last 24 hours:**
```kql
EmailPostDeliveryEvents
| where Timestamp > ago(24h)
| where ActionType in ("Phish ZAP", "Malware ZAP")
| join kind=inner EmailEvents on NetworkMessageId
| project Timestamp, ActionType, SenderFromAddress, RecipientEmailAddress, Subject
| order by Timestamp desc
```

**Top senders generating phishing detections (last 14 days):**
```kql
EmailEvents
| where Timestamp > ago(14d)
| where ThreatTypes has "Phish"
| summarize DetectionCount = count() by SenderFromAddress, SenderIPv4, SenderMailFromDomain
| order by DetectionCount desc
| take 25
```

**Users who clicked on URLs detected as malicious:**
```kql
UrlClickEvents
| where Timestamp > ago(7d)
| where ActionType == "ClickAllowed"
| where ThreatTypes has "Phish"
| project Timestamp, AccountUpn, Url, IsClickedThrough
| order by Timestamp desc
```

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-emailevents-table
https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-emailattachmentinfo-table
https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-emailurlinfo-table
https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-emailpostdeliveryevents-table
https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-urlclickevents-table
```

</details>

---

<details>
<summary><h2>9. Troubleshooting</h2></summary>

> 🧭 **Navigation:** `admin.exchange.microsoft.com` → Mail flow → **Message trace**

### General Approach to troubleshooting

1. **Run a message trace** — Identifies delivery status, rejection reason, and routing path
2. **Check policy priority order** — Confirm the expected policy applied to the recipient
3. **Review detection details in Threat Explorer** — Shows which signal fired and why
4. **Check TABL for conflicting entries** — A block and allow for the same entity can cause unexpected behavior
5. **Check the Submissions page** — If an allow was submitted, confirm it was accepted and applied

---

### Common Issues and Resolutions

| Issue | Likely Cause | Resolution |
|-------|-------------|------------|
| Legitimate email quarantined | Policy threshold too aggressive or no allow entry | Submit via Submissions page, add TABL allow if needed |
| Phishing email delivered | Indicator not in TABL; MDO stack did not detect | Investigate detection gap; submit as FN; promote indicator |
| TABL block entry not taking effect | Entry not yet propagated (allow up to 5 minutes) | Wait and retest; verify entry syntax is correct |
| Block entry syntax rejected | Invalid format for entry type | Review URL or domain syntax requirements; wildcards follow specific rules |
| Allow entry for malware not accepted | Not supported directly in TABL | Use Submissions page to report and request allow |
| ZAP did not remediate a delivered email | ZAP has a 48-hour post-delivery window | For older mail, use manual remediation via Threat Explorer |
| Automation playbook failed to add TABL entry | Permissions or throttling | Check Exchange Online PowerShell credentials and retry logic in playbook |

---

### Message Trace

Use message trace to confirm what happened to a specific email.

```
admin.exchange.microsoft.com → Mail flow → Message trace → New trace
```

Message trace provides:
- Delivery status and reason
- Which mail flow rules applied
- Which connectors were used
- Policy-level filtering decisions

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/run-a-message-trace-and-view-results
https://learn.microsoft.com/en-us/defender-office-365/anti-spam-protection-faq
https://learn.microsoft.com/en-us/defender-office-365/zero-hour-auto-purge
```

</details>

---

<details>
<summary><h2>9b. Attack Simulation Training</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Email & collaboration → **Attack simulation training**

Attack Simulation Training (MDO Plan 2 or Microsoft 365 E5) is a customer-run capability to measure phishing resilience and deliver targeted training.

### When to Use It

- Deliver compliance-driven periodic training (quarterly cadence is common).
- Measure improvement after targeted training or policy changes.

### Simulation Types

| Technique | What It Tests |
|---|---|
| Credential harvest | User susceptibility to fake sign-in pages |
| Malware attachment | Execution of weaponized documents (detonated in sandbox) |
| Link in attachment | Following links embedded in attached files |
| Link to malware | Downloading from a linked payload |
| Drive-by URL | Engagement with malicious URLs |
| OAuth consent grant | Consent to malicious applications |

### Deployment Steps

1. **Assign roles** — *Attack Simulator Administrator* or *Security Administrator*.
2. **Configure simulation settings** — allow-list simulation sending IPs in the connection filter and exclude simulation domains from policy filtering where required (use the managed allowlist via Threat policies → *Advanced delivery* → **Phishing simulation**).
3. **Pilot a simulation** — start with a small scoped group, moderate difficulty, to validate delivery and reporting before going broad.
4. **Launch simulation** — production-wide, moderate difficulty, measure compromise rate and reporting rate.
5. **Assign training** — auto-assign Microsoft-authored modules to users who failed the simulation.
6. **Track trend** — monitor compromise and reporting rates over time in the Simulation and Training dashboards.

### Operational Guidance

- Coordinate simulation windows with the SOC so reported messages are not treated as live incidents.
- Use **Advanced delivery configuration** (Threat policies → Advanced delivery → *Phishing simulation*) to register approved third-party simulation vendors as well, so their messages are not blocked by MDO.
- Avoid simulating during major business events or incident response periods.

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-office-365/attack-simulation-training-simulations
https://learn.microsoft.com/en-us/defender-office-365/advanced-delivery-policy-configure
```

</details>

---

<details>
<summary><h2>10. Content Filtering</h2></summary>

> 🧭 **Navigation:** `security.microsoft.com` → Email & collaboration → Policies & rules → Threat policies → **Anti-spam**
> Also configurable via Exchange Online PowerShell

### What Is Content Filtering?

Content filtering assigns a Spam Confidence Level (SCL) to inbound email based on content analysis — Microsoft's Intelligent Message Filter (IMF) heuristics, machine-learning models, and customer-defined overrides.

> ⚠️ **Important platform note:** The legacy `Add-ContentFilterPhrase` / `Get-ContentFilterPhrase` / `Remove-ContentFilterPhrase` cmdlets (GoodWord / BadWord phrase lists) are **available only in on-premises Exchange**. They are **not** available in Exchange Online. In Exchange Online / Defender for Office 365, use the controls described below instead.
>
> In Exchange Online, keyword-based SCL overrides are implemented with **mail flow (transport) rules** that set or modify the **SCL** header. Sender-level overrides use the **Allowed / Blocked senders and domains** lists inside each anti-spam policy.

Content filtering is **not the same as TABL** — TABL acts on structured indicators (sender, URL, file hash, IP, spoof). Content filtering (and transport-rule SCL overrides) act on message content.

💡 **Choosing the right control:**
- Sender or domain exceptions → anti-spam policy **Allowed / Blocked senders and domains**
- Keyword / phrase / regex matching in subject, body, or headers → **Mail flow rules** with *Set the SCL to…* action
- Tenant-wide indicator overrides (sender, URL, file hash, IPv6, spoof) → **TABL**

---

### How SCL Is Determined in Exchange Online

```
Inbound message received
    ↓
Connection filter, authentication, anti-malware, anti-phish
    ↓
Mail flow rules evaluated — a rule can set SCL to -1 (bypass) or 5/6/9 (force spam verdict)
    ↓
If no rule sets SCL, MDO/EOP content analysis assigns an SCL (0, 1, 5, 6, 9)
    ↓
Anti-spam policy action applied based on final SCL verdict
  → SCL 5–6: Spam action (move to junk or quarantine)
  → SCL 7–9: High confidence spam action (quarantine)
```

---

### Mail Flow Rule Limits (Exchange Online)

These limits apply to mail flow rules used to implement keyword/phrase-based SCL overrides in EXO:

| Limit | Value | Notes |
|-------|-------|-------|
| Rules per tenant | 300 | Hard service limit |
| Size per rule | 8 KB | Covers conditions, exceptions, and actions |
| Total regex across all rules | 20 KB | Org-wide limit |
| Case sensitivity (word match) | Case-insensitive | "Risk" matches "RISK", "risk", "RiSk" |
| Wildcards / regex | ✅ Supported | Use *matches patterns* conditions for regex |

> ℹ️ The on-premises Content Filter agent limits (800 GoodWord/BadWord phrases, 11 MB message ceiling, 2 MB attachment extract, etc.) do **not** apply to Exchange Online. Those limits are documented in Exchange Server content.

---

### How to Configure Content-Based SCL Overrides in Exchange Online

#### Prerequisites
- Exchange Online mailboxes (cloud). On-premises Exchange admins use `Add-ContentFilterPhrase` — not covered here.
- Role: **Security Administrator** (for anti-spam policies) or **Exchange Administrator / Transport Rules role** (for mail flow rules).

#### Option 1 — Keyword / phrase rules (replaces BadWord / GoodWord)

**Portal:** Exchange admin center (`admin.exchange.microsoft.com`) → Mail flow → **Rules** → *+ Add a rule* → *Create a new rule*.

Example — force high-confidence spam on messages containing a known lure phrase:

| Field | Value |
|---|---|
| Apply this rule if | The subject or body **includes any of these words** |
| Words | `urgent wire transfer`, `verify your account immediately` |
| Do the following | **Modify the message properties** → *set the spam confidence level (SCL) to* → **9** |
| Except if | Sender is in the org's allowed-sender list (optional) |

Example — bypass spam filtering for internal/partner traffic matching a signed keyword (rare; use sparingly):

| Field | Value |
|---|---|
| Apply this rule if | The sender is located **Outside the organization** AND a message header matches a known partner signing header |
| Do the following | Set the SCL to **-1** |

> ⚠️ SCL `-1` bypasses spam filtering only. Malware and high-confidence phishing are still evaluated by **Secure by Default** and cannot be bypassed this way.

Equivalent Exchange Online PowerShell:
```powershell
# Block — force SCL 9 on a keyword match
New-TransportRule -Name "Block keyword - urgent wire transfer" `
  -SubjectOrBodyContainsWords "urgent wire transfer","verify your account immediately" `
  -SetSCL 9

# Allow — set SCL -1 (bypass spam filtering; malware/HC phish still enforced)
New-TransportRule -Name "Allow partner - signed header" `
  -HeaderMatchesMessageHeader "X-Partner-Signed" `
  -HeaderMatchesPatterns "^true$" `
  -SetSCL -1
```

#### Option 2 — Sender / domain allow and block lists (per anti-spam policy)

**Where:** `security.microsoft.com` → Email & collaboration → Policies & rules → Threat policies → **Anti-spam** → edit policy → **Allowed and blocked senders and domains**.

Use for per-policy sender exceptions, not keyword matching. These lists do **not** override **Secure by Default** for malware or high-confidence phishing.

#### Option 3 — Tenant Allow/Block List (TABL)

Use for tenant-wide indicator enforcement (senders, URLs, files, IPv6, spoof). See Section 2.

#### Verify anti-spam policy actions take the SCL correctly

```powershell
Get-HostedContentFilterPolicy | Format-List Name,SpamAction,HighConfidenceSpamAction
```

Recommended values:
- `SpamAction`: `MoveToJmf` or `Quarantine`
- `HighConfidenceSpamAction`: `Quarantine`


---

### Operational Notes

- **Propagation time:** Changes to mail flow rules and anti-spam policies take effect within minutes.
- **Match precision:** Transport rule *includes any of these words* matches whole words, case-insensitive. For fuzzy or pattern matches (e.g., `wire-transfer`, `wiretransfer`), use *matches these text patterns* with regex.
- **SCL override behavior:** A transport-rule `SetSCL 9` forces high-confidence spam treatment. `SetSCL -1` bypasses spam filtering only — **Secure by Default** still enforces malware and high-confidence phishing quarantine.
- **Review cadence:** Include mail flow rule review in the monthly operations checklist (Section 7) to remove stale or no-longer-relevant rules.

---

### 📚 Reference Articles
```
https://learn.microsoft.com/en-us/defender-office-365/anti-spam-protection-about
https://learn.microsoft.com/en-us/defender-office-365/anti-spam-policies-configure
https://learn.microsoft.com/en-us/defender-office-365/anti-spam-policies-asf-settings-about
https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules
https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/conditions-and-exceptions
```

</details>

<details>
<summary><h2>11. Resources</h2></summary>

### Portal and Operations Notes

### Core MDO Documentation
```
https://learn.microsoft.com/en-us/defender-office-365/
```

### Preset Security Policies
```
https://learn.microsoft.com/en-us/defender-office-365/preset-security-policies
```

### Tenant Allow/Block List
```
https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-about
```

### Threat Explorer
```
https://learn.microsoft.com/en-us/defender-office-365/threat-explorer-about
```

### Submissions
```
https://learn.microsoft.com/en-us/defender-office-365/submissions-admin
```

### Sentinel Integration
```
https://learn.microsoft.com/en-us/azure/sentinel/microsoft-365-defender-sentinel-integration
```

### Advanced Hunting Schema
```
https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-schema-tables
```

### Zero-Hour Auto Purge (ZAP)
```
https://learn.microsoft.com/en-us/defender-office-365/zero-hour-auto-purge
```

### MDO SecOps Operations Guide
```
https://learn.microsoft.com/en-us/defender-office-365/mdo-sec-ops-guide
```

### MDO Step-by-Step Protection Stack
```
https://learn.microsoft.com/en-us/microsoft-365/security/office-365-security/protection-stack-microsoft-defender-for-office365
```

### Graph API Security Overview
```
https://learn.microsoft.com/en-us/graph/api/resources/security-api-overview
```

### **MDO Preset Policy Documentation**
- Preset security policies overview and Standard/Strict behavior:
  - https://learn.microsoft.com/en-us/defender-office-365/preset-security-policies
- Recommended settings tables (Default vs Standard vs Strict):
  - https://learn.microsoft.com/en-us/defender-office-365/recommended-settings-for-eop-and-office365
- Deployment guidance (policy strategy and rollout):
  - https://learn.microsoft.com/en-us/defender-office-365/mdo-deployment-guide
</details>
---

*Last updated: April 2026*
