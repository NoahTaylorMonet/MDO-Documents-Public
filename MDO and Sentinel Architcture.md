# MDO and Sentinel Architecture Recommendation

<details open>
<summary><strong>Executive Briefing</strong></summary>

## Executive Briefing

### Decision Requested

Adopt a Microsoft-native architecture in which:

- **Microsoft Sentinel** becomes the long-term repository and correlation layer for the full Boeing email indicator corpus.
- **Microsoft Defender for Office 365 (MDO)** is used as the selective enforcement plane for active, high-confidence indicators.
- **Automation** handles promotion into enforcement, downstream remediation, and expiration of dormant indicators.

### Why This Decision Is Needed

Boeing currently manages approximately **290,000 email-related indicators**. Microsoft-native enforcement controls are effective, but they are not intended to store the full indicator corpus indefinitely. Directly placing the entire corpus into MDO enforcement surfaces would create unnecessary operational strain and would not align with product limits.

### Recommended Outcome

This architecture gives Boeing:

- Scalable indicator retention in Sentinel
- Selective enforcement in MDO for the indicators that matter most operationally
- Better automation and reduced manual list management
- Clear analyst visibility across detection, enforcement, and remediation
- A design that fits Microsoft platform constraints rather than working against them

### Executive-Level Implementation Expectation

- **Estimated elapsed timeline:** 10 to 18 weeks
- **Primary teams involved:** Security architecture, Sentinel engineering, MDO engineering, SOC operations, and platform automation owners

### Executive Risk Summary

The largest risks are not product capability gaps; they are implementation quality issues:

- Overly aggressive promotion logic that fills enforcement controls too quickly
- Insufficient exception handling in automation
- Lack of operational ownership for tuning and expiration logic

These risks are manageable with phased rollout, pilot validation, and explicit engineering guardrails.
</details>

---

<details>
<summary><strong>Architecture Recommendation</strong></summary>

## Architecture Recommendation

Boeing should adopt the following target-state model:

- **Microsoft Sentinel** stores and correlates the full indicator inventory at enterprise scale.
- **Microsoft Defender for Office 365** is used for selective enforcement of active, high-confidence indicators.
- **Automation** is used to promote indicators into enforcement, trigger response actions, and remove stale indicators after a defined dormancy period.
- **Analyst workflows** remain centered on Sentinel, Threat Explorer, and XDR for triage, explanation, and response.

This recommendation avoids trying to force the full Boeing indicator corpus directly into enforcement controls that were not designed to hold hundreds of thousands of entries simultaneously.
</details>

---

<details>
<summary><strong>Why This Architecture Is Needed</strong></summary>

## Why This Architecture Is Needed

Boeing currently manages approximately 290,000 email-related indicators from multiple internal and external intelligence providers. MDO provides strong protection, but customer-managed allow/block features have finite limits and should be used selectively.

If Boeing attempts to use direct enforcement lists as the primary long-term repository for the full indicator set, the result will be unnecessary operational complexity, frequent curation pressure, and reduced flexibility. A Sentinel-led architecture solves this by separating:

- **Intelligence retention and correlation** from
- **Real-time enforcement**

That separation allows Boeing to keep broad intelligence coverage without exhausting the smaller, higher-value enforcement surfaces in MDO.
</details>

---

<details>
<summary><strong>Protection Model</strong></summary>

## Protection Model

It is important to note that Microsoft Defender for Office 365 protection is not limited to indicators explicitly placed in the Tenant Allow/Block List (TABL).

Even when an indicator is not present in TABL, MDO continues to inspect messages through its broader protection stack. Messages are still evaluated using filtering, reputation, impersonation detection, anti-phishing, malware analysis, and detonation logic. If a dormant indicator reappears and triggers suspicious activity, MDO can still quarantine, block, or detonate the message based on risk signals.

That telemetry can then be ingested into Microsoft Sentinel, where it can be correlated and, if appropriate, automatically promoted into MDO enforcement for subsequent activity.

This means Boeing does **not** need every indicator loaded into TABL to maintain protection. TABL should instead be reserved for the subset of indicators that are:

- Currently active
- High confidence
- Operationally relevant for immediate enforcement
</details>

---

<details>
<summary><strong>Content Filtering Role in This Architecture</strong></summary>

## Content Filtering Role in This Architecture

### What It Is

Content filtering is Microsoft 365 anti-spam text inspection that evaluates message content (subject, headers, and body) and applies spam actions based on policy verdicts.

### How It Works in the Boeing Model

In this architecture, content filtering is a **targeted supplemental control**:

- Use it for specific high-value keyword/phrase scenarios.
- Keep Sentinel as the intelligence hub and TABL as the primary high-confidence indicator enforcement cache.
- Use transport rules when Boeing needs more granular or regex-based matching logic.

### Practical Limits (Executive View)

- Phrase-based control is suitable for focused business/security cases, not enterprise-scale indicator repositories.
- Microsoft documentation cites finite phrase and rule capacities; therefore this control should remain scoped and curated.
- Content inspection is bounded by message and attachment scanning constraints, so it should not be treated as a universal deep-inspection layer.

### What It Scans (At a Glance)

- Subject and message body
- Message headers
- Supported attachment content (within Microsoft scanning limits)

### Recommended Governance

- Maintain a small, approved phrase set owned by SOC + security architecture.
- Apply monthly review and expiration for phrase entries.
- Route complex matching needs to transport rules or Sentinel-driven workflows.

### How to Turn It On / Configure

Implementation steps are documented in the deployment playbook:

1. Open anti-spam policy settings in Defender.
2. Configure content filtering behavior and actions.
3. Add or remove approved phrase controls as needed.
4. Validate outcomes in message trace, Threat Explorer, and SOC workflows.

See the detailed runbook in the MDO deployment guide (Content Filtering section) for operational commands and policy-level configuration.

</details>

---
<details open>
<summary><strong>Target-State Architecture</strong></summary>

## Target-State Architecture

### Core Design

- **Sentinel as the intelligence hub**  
  Sentinel ingests and stores the full indicator set from internal and external sources. It correlates those indicators with email telemetry, Defender XDR signals, message metadata, URLs, attachments, headers, and behavioral detections.

- **MDO as the enforcement plane**  
  MDO enforces a curated subset of indicators using native controls such as the Tenant Allow/Block List and other mail protection policies.

- **Automation as the promotion and response mechanism**  
  Analytics and playbooks determine when an indicator becomes active, when it should be promoted into enforcement, what response actions should occur, and when it should be removed from enforcement after dormancy.

### Recommended Processing Flow

1. Ingest indicators into Sentinel from Boeing and third-party intelligence sources.
2. Correlate indicators against MDO, XDR, and email telemetry.
4. Detect indicators that become active or rise above Boeing-defined confidence thresholds.
5. Promote qualifying indicators into MDO enforcement controls.
6. Trigger automated response actions such as message removal, case creation, notifications, or endpoint actions.
7. Age out dormant indicators from direct enforcement while retaining them in Sentinel for continued monitoring.

### Engineering Design Principles

- Treat **Sentinel** as the source of truth for indicator state.
- Treat **MDO** as a constrained enforcement cache, not a master repository.
- Promotion logic should be **deterministic, explainable, and reversible**.
- Every automation step should produce **observable logs and auditable state changes**.
- Enforcement capacity should be monitored continuously to avoid unexpected saturation.
</details>

---

<details>
<summary><strong>MDO Limits and Engineering Constraints</strong></summary>

## MDO Limits and Engineering Constraints

### TABL Limits

| Category | Feature / Limit | MDO Plan 2 |
|---|---|---|
| TABL - Domains & Email Addresses | Total entries per tenant | 15,000 (5,000 allow / 10,000 block) |
| TABL - URLs | Total entries per tenant | 15,000 (5,000 allow / 10,000 block) |
| TABL - Files (SHA256 hashes) | Total entries per tenant | 15,000 (5,000 allow / 10,000 block) |
| TABL - IP Addresses (IPv6 only) | Total entries per tenant | 15,000 (5,000 allow / 10,000 block) |
| TABL - Spoofed Senders | Total entries (allow + block combined) | 1,024 combined |

### TABL Notes

- The Tenant Allow/Block List IP addresses tab currently supports **IPv6 only**.
- **IPv4 support for TABL IP addresses is being worked on by engineering and is currently planned for next quarter, subject to change.**
- Until that capability ships, IPv4 addresses and ranges are **not** supported in TABL.
- TABL entries typically become active within about **5 minutes**.
- TABL IP allow entries bypass **IP-based filtering checks** such as connection filtering or IP reputation checks, but they do **not** change throttling behavior.
- TABL IP block entries reject mail **at the service edge**.

### Non-TABL MDO / Exchange Limits

| Category | Feature / Limit | Limit |
|---|---|---|
| Connection Filter | IP Allow List | 1,273 entries |
| Connection Filter | IP Block List | 1,273 entries |
| Anti-Spam Policy | Allowed (safe) sender list per policy | 1,024 entries |
| Anti-Spam Policy | Blocked sender list per policy | 1,024 entries |
| Exchange Transport Rules | Maximum rules per tenant | 300 rules |

### Non-TABL Control Notes

- **Connection filter policies support IPv4** and remain the primary Microsoft-native control for IPv4 IP allow and block use cases today.
- **Anti-spam policies support IPv4-based mail flows operationally**, but their configurable controls are sender and domain based rather than IP-list based.
- Connection filter entries can be a **single IP**, an **IP range**, or a **CIDR range**.
- For connection filtering CIDR entries, supported masks are **/24 through /32**.
- The connection filter **IP Allow List** takes precedence over the **IP Block List**.
- Messages from sources in the connection filter IP Allow List still receive **malware** and **high confidence phishing** inspection.
- The default connection filter policy also includes a Microsoft-managed **safe list** capability for known good senders.

### Engineering Guidance

The engineering teams should treat these limits as **hard platform constraints** and design automation accordingly:

- Add pre-write checks before promotion into TABL.
- Reject or defer promotions if capacity thresholds are exceeded.
- Reserve capacity for emergency response actions.
- Avoid using transport rules as a primary scaling mechanism for indicator enforcement.
- Keep promotion logic type-aware so that domains, URLs, files, spoofed senders, and IPs are each routed to the correct control.
</details>

---

<details>
<summary><strong>Engineering Implementation Model</strong></summary>

## Engineering Implementation Model

### Data Model Expectations

Each indicator should carry, at minimum, the following metadata in Sentinel:

- Indicator value
- Indicator type
- Source provider
- Confidence / severity
- First seen timestamp
- Last seen timestamp
- Expiration or review date
- Promotion eligibility status
- Current enforcement status
- Last enforcement action time
- Automation notes or failure state

### Recommended Logical Components

- **Ingestion layer** for external and internal threat feeds
- **Correlation layer** in Sentinel analytics
- **Decision layer** for promotion and expiration logic
- **Enforcement integration layer** for MDO actions
- **Response orchestration layer** for incident creation and remediation
- **Monitoring layer** for health, failures, and capacity tracking

### Recommended Automation Controls

Automation should include:

- Capacity checks before adding indicators to MDO
- Duplicate detection before writing new entries
- Exception queues for indicators that fail validation or exceed thresholds
- Rollback logic for failed writes or accidental over-promotion
- Audit logs for every create, update, and remove action
- Alerting on failed playbooks or unexpected growth in promoted indicators

### Recommended Approval Model

A practical starting model is:

- **Automatic promotion** for indicators that exceed a high-confidence threshold and match confirmed activity
- **Analyst approval** for edge cases, ambiguous sources, or large-volume changes
- **Automatic expiration** after inactivity thresholds are met

This keeps execution fast without removing human judgment where it still matters.
</details>

---

<details open>
<summary><strong>Implementation Plan and Timeline</strong></summary>

## Implementation Plan and Timeline

The implementation should be delivered in phases rather than as a single cutover. That reduces operational risk and gives Boeing time to tune thresholds, validate automation, and measure enforcement volume before expanding the model.

### Phase 1: Architecture and Design

**Objective:** Define the operating model, data sources, indicator taxonomy, promotion criteria, and enforcement boundaries.

**Activities:**

- Confirm authoritative indicator sources
- Define indicator types, schemas, confidence levels, and metadata requirements
- Define promotion criteria for moving indicators from Sentinel into MDO enforcement
- Define dormancy and expiration logic
- Define analyst workflows, escalation paths, and ownership boundaries
- Confirm reporting and audit requirements

**Estimated duration:** 1 to 2 weeks

- Primary owners: **Security architecture, SOC leads, platform engineering**

### Phase 2: Sentinel Ingestion and Data Modeling

**Objective:** Build the ingestion pipeline and validate that indicators from Boeing and third-party sources land correctly in Sentinel with the expected schema, metadata, and retention properties.

**Activities:**

- Create and validate ingestion paths for internal and external indicator feeds
- Confirm source attribution, timestamps, confidence, and retention metadata are preserved correctly
- Build or validate analytics-ready tables and queries in Sentinel

**Estimated duration:** 2 to 4 weeks

- Primary owners: **Sentinel engineering, integration engineering**

### Phase 3: Detection and Correlation Logic

**Objective:** Determine how indicators become eligible for action.

**Activities:**

- Build analytics rules that correlate indicators with email and XDR telemetry
- Define thresholds for active versus dormant indicators
- Tune logic to reduce noise and avoid unnecessary promotion into MDO
- Validate sample detections against historical incidents or known bad traffic

**Estimated duration:** 2 to 3 weeks

- Primary owners: **Detection engineering, SOC engineering**

### Phase 4: Automation and Enforcement Integration

**Objective:** Automate promotion into MDO and automate downstream response actions.

**Activities:**

- Build automation to add qualifying indicators into TABL or other applicable controls
- Build automation to remove dormant indicators after the defined window
- Integrate response actions such as ticketing, mailbox cleanup, notifications, or endpoint actions where appropriate
- Add logging, exception handling, and rollback safeguards
- Test propagation timing and failure handling

**Estimated duration:** 1 to 2 weeks

- Primary owners: **Automation engineering, MDO engineering, SOC automation owners**

### Phase 5: Pilot and Tuning

**Objective:** Run the model in a controlled scope before broad production adoption.

**Activities:**

- Pilot on a subset of indicators or business units
- Measure promotion volume, false positives, and analyst workload
- Tune confidence thresholds and dormancy windows
- Validate reporting and incident workflows
- Confirm that enforcement volumes stay safely inside MDO limits

**Estimated duration:** 2 to 3 weeks

- Primary owners: **SOC operations, engineering leads, threat operations**

### Phase 6: Production Rollout and Operations Handover

**Objective:** Move the architecture into steady-state operations.

**Activities:**

- Roll out production automation
- Document operational runbooks
- Define ownership for maintenance and exception handling
- Establish KPI and limit monitoring
- Conduct knowledge transfer to analysts and administrators

**Estimated duration:** 1 to 2 weeks

- Primary owners: **Operations, platform owners, SOC leadership**

### Overall Timeline

For an average implementation, Boeing should expect an elapsed delivery timeline of approximately **10 to 18 weeks** across all phases.

Actual duration will vary based on:

- The quality and consistency of upstream indicator feeds
- Existing Sentinel maturity
- Existing automation standards and approval requirements
- Required integrations such as ServiceNow, ticketing, mailbox remediation, or custom approval workflows
- Boeing's tolerance for auto-enforcement versus analyst approval steps
</details>

---

<details>
<summary><strong>Analyst Operations and Runtime Expectations</strong></summary>

## Analyst Operations and Runtime Expectations

This architecture preserves strong analyst visibility and investigative depth.

Analysts can use:

- **Microsoft Sentinel** for large-scale correlation, hunting, automation monitoring, and incident generation
- **Threat Explorer** for message-level investigation and delivery analysis
- **Microsoft Defender XDR pivots** for cross-domain investigation across identities, endpoints, and email activity

This makes it possible to answer questions such as:

- Why was a message blocked?
- Which indicator triggered enforcement?
- Was the action the result of TABL, connection filtering, policy logic, or broader MDO detection?
- What remediation action has already occurred?

Policy and indicator updates may take several minutes to propagate through automation and platform controls. Based on current expectations, average propagation should often fall in the **3 to 5 minute** range, though in some cases it may take **up to 15 minutes**.

### Steady-State Operational Expectations

For a mature deployment, ongoing operational demand should be low. Most promotion and removal actions will be automated, with human effort focused on:

- Recurring review of automation health and exception queues
- Periodic threshold and tuning adjustments
- Exception handling for disputed blocks or failed automation runs
</details>

---

<details>
<summary><strong>Engineering Execution Notes</strong></summary>

## Engineering Execution Notes

To execute this architecture cleanly, engineering teams should align on the following practical expectations:

- Build for **idempotent automation** so retries do not create inconsistent state.
- Use a **single authoritative promotion state** in Sentinel to avoid split-brain logic between systems.
- Monitor **capacity consumption by indicator type** and alert before hard limits are reached.
- Separate **feed ingestion failures** from **enforcement failures** so operational triage is faster.
- Require **structured logging** for every promotion, expiration, and remediation action.
- Maintain an **exception path** for indicators that cannot be enforced due to type, syntax, or capacity constraints.
- Validate control mappings early, especially for **IPv4 vs IPv6**, **spoofed sender pairs**, and **file-hash submission behavior**.
- Keep a **pilot-safe mode** in which automation recommends or stages changes before writing them.

A weak implementation will fail because of process ambiguity and poor state handling, not because Sentinel or MDO are incapable. The engineering focus should therefore be on correctness, traceability, and operational resilience.
</details>

---

<details>
<summary><strong>Conclusion</strong></summary>

## Conclusion

The recommended architecture is to use Microsoft Sentinel as the long-term intelligence and correlation platform, and Microsoft Defender for Office 365 as the selective enforcement layer for actively observed, high-confidence threats.

This is the most scalable and operationally sound design for Boeing's indicator volume. It respects Microsoft platform limits, maintains layered protection, supports automation-led response, and gives analysts a practical model for investigation and enforcement without trying to force the entire indicator corpus into direct enforcement controls.
</details>
