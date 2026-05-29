# MDO and Sentinel Indicator Deployment Guide

> **Purpose:** This guide details the indicator lifecycle from ingestion into Sentinel, through promotion to MDO, and enforcement and eventual expiration.

---

## Executive Summary

Organizations that handle large indicator volumes need both analytics depth and enforcement precision. This guide uses Microsoft Sentinel and Microsoft Defender for Office 365 (MDO) together so each platform does what it is best at:

- Sentinel is the aggregation, normalization, correlation, and decision layer for high-volume indicator data.
- MDO is the enforcement layer that applies validated indicators to the Tenant Allow/Block List (TABL).

### Why Use Them Together

- Scale: Sentinel can ingest and process large indicator sets from multiple sources.
- Context: Sentinel correlates indicators with security telemetry to reduce blind promotion.
- Control: MDO enforces only vetted, high-confidence indicators.
- Lifecycle: Automation supports promotion, monitoring, expiration, and re-promotion as activity changes.

### How the Combined Model Works

1. Ingest indicators into Sentinel from approved internal and external sources.
2. Correlate indicators with relevant telemetry (for example, email and click activity).
3. Apply promotion criteria to select indicators.
4. Promote eligible indicators to MDO TABL through playbook.
5. Monitor, track capacity, and handle exceptions.
6. Expire stale indicators and re-promote only when new evidence appears.

---
## 1. Sentinel Ingestion
<details>
<summary><strong>Implementation</strong></summary>

### Implementation Options

**Option 1: Sentinel STIX Objects Upload API (Recommended)**
- Build a Logic App or custom app that reads your indicator source and uploads STIX 2.0/2.1 objects to Sentinel Upload API
- Endpoint: https://api.ti.sentinel.azure.com/workspaces/{workspaceId}/threat-intelligence-stix-objects:upload?api-version=2024-02-01-preview
- Authentication: Microsoft Entra app with **Microsoft Sentinel Contributor** role at the workspace scope and token audience https://management.azure.com/.default
- Request body fields: sourcesystem, stixobjects
- Throttling limits: 100 objects/request and 100 requests/minute

**Option 2: Threat Intelligence - TAXII Connector**
- Use Sentinel's TAXII 2.x connector when your feed is already exposed via STIX/TAXII server collections
- Maintenance: Lower if provider has stable TAXII implementation

**Option 3: Custom Data Connector**
- Build custom connector using Azure Functions + App Service for non-STIX/TAXII sources
- Maintenance: Higher (custom code, versioning, updates)

> **[UPDATED - BEST PRACTICE]** For TI ingestion into Sentinel, use the Upload API (STIX Objects API). Legacy TI ingestion paths based on older TI connector patterns are on deprecation path.

### Ingestion Checklist

- [ ] Microsoft Entra app registration created and granted **Microsoft Sentinel Contributor** on the target workspace
- [ ] Upload API endpoint configured (pi-version=2024-02-01-preview)
- [ ] Ingestion payload mapped to STIX 2.0/2.1 (sourcesystem + stixobjects)
- [ ] Ingestion frequency defined (hourly / 4x daily)
- [ ] Logic App, TAXII connector, or custom connector built and tested
- [ ] Throttling and retry logic implemented (respect 100 objects/request, 100 requests/min)

</details>

---

## 2. Indicator Correlation Using Sentinel Analytic Rules

<details>
<summary><strong>Linking Indicators to Mailbox Activity</strong></summary>      

Once indicators are ingested, Sentinel analytic rules handle correlation. Out-of-box templates can accelerate implementation, but custom KQL may be required for organization-specific indicator models, telemetry mappings, and tuning.

</details>

---

## 3. Promotion to TABL via Graph API

<details>
<summary><strong>Moving Indicators from Sentinel to MDO Enforcement</strong></summary>

High-confidence indicators are promoted to TABL using Microsoft Graph API.      

### TABL Capacity by Indicator Type

| Type | TABL Tab | Limit | Current Support |
|------|----------|-------|-----------------|
| Domain/Email | Domains & Addresses | 15,000 | Active |
| URL | URLs | 15,000 | Active |
| File Hash (SHA256) | Files | 15,000 | Active |
| IPv6 Address | IP Addresses | 15,000 | Active |
| Spoofed Sender | Spoofed Senders | 1,024 | Active |
| IPv4 Address | IP Addresses | N/A | Roadmap (next quarter) |


### Building, Testing, Validation, and Production Setup

This section provides a practical sequence your organization can follow to build confidence before full-scale deployment.

#### Build Scope

Start with a constrained pilot that proves end-to-end lifecycle behavior for a small set of indicators.

**Pilot Objectives:**
- Validate ingestion from one internal source and one external source into Sentinel
- Validate Sentinel analytic rule logic against MDO/XDR email telemetry
- Validate promotion logic into TABL for high-confidence active indicators      
- Validate expiration/demotion logic for dormant indicators
- Validate analyst visibility and auditability across each transition

**Pilot Size Recommendation:**
- 200 to 1,000 indicators
- At least 4 indicator types (domain/address, URL, file hash, spoofed sender)   

#### Build Steps

1. **Create new automation for pilot**
   - Create dedicated analytics rules/playbooks (do not reuse production playbooks)
   - Add explicit naming convention (for example: \POC-Indicator-*\)

2. **Define Data in Sentinel**
   - Required fields: indicator value, type, confidence, source, first seen, last seen, status
   - Add lifecycle state field: \Observed\ -> \Promoted\ -> \Expired\

3. **Implement promotion policy**
   - Example criteria: confidence >= your organization's configured threshold + active observation in last N days

4. **Error checks**
   - Capacity checks before TABL write
   - Duplicate detection
   - Retry/backoff for API failures
   - Exception queue for manual analyst review

5. **Auditing**
   - Write all promotion and expiration actions to a dedicated log table        
   - Capture reason codes (why promoted, why rejected, why expired)

#### Testing and Validation

**Phase A: Functional Validation**
- [ ] Indicator is ingested into Sentinel with correct schema
- [ ] Analytic rule identifies matching MDO/XDR activity
- [ ] Eligible indicator is promoted to correct TABL type
- [ ] Non-eligible indicator is held (not promoted)
- [ ] Duplicate indicator does not create duplicate TABL entry

**Phase B: Negative and Exception Testing**
- [ ] TABL capacity threshold breach triggers deferral/alert behavior
- [ ] API timeout or throttling triggers retry       
- [ ] Unknown routed for analyst review

**Phase C: Operational Validation**
- [ ] Analysts can trace indicator state transitions end-to-end
- [ ] SOC can explain promotion decision from logs alone
- [ ] Logic removes stale TABL entries as expected
- [ ] Alerting works for failed automation actions

**Phase D: Security and Governance Validation**
- [ ] Service principals/use identities follow least privilege
- [ ] Change controls exist for promotion thresholds
- [ ] Audit trail is retained for compliance window


#### Production Setup Steps

1. **Establish production guardrails**
   - Set per-type TABL capacity thresholds
   - Reserve emergency headroom for incident response
   - Define maximum daily promotion volume limits

2. **Promote infrastructure to production**
   - Deploy production playbooks, identities, secret handling, and monitoring   
   - Separate prod and non-prod automation identities and logging workspaces    

3. **Enable phased production rollout**
   - Week 1: domains/addresses only
   - Week 2: add URLs
   - Week 3: add file hashes
   - Week 4: add spoofed senders
   - Expand only after each phase meets error-rate/SOC acceptance thresholds    

4. **Activate production monitoring**
   - Dashboards for promotions, expirations, failures, capacity consumption, and mean time to recovery
   - Alerts for failed writes, throttling spikes, and abnormal promotion surges 

5. **Run production validation checkpoint (first 14 days)**
   - Daily engineering + SOC review
   - Confirm no runaway promotions
   - Confirm false-positive and exception handling operates as designed

6. **Transition to steady-state operations**
   - Move to weekly governance review
   - Monthly threshold and exclusion review
   - Quarterly assessment


### What This Looks Like in Action

In practice, Sentinel does not promote every observed indicator. A Sentinel analytic rule identifies a candidate, the workflow checks policy and TABL capacity, and only then sends the approved indicator to MDO for enforcement. (TI ingestion into Sentinel should use STIX Objects Upload API.)

Typical promotion flow:

1. Sentinel identifies a high-confidence indicator such as attack.com.
2. The workflow checks confidence, recent activity, duplicate status, and target TABL type.
3. Automation submits the indicator to the correct MDO TABL section.
4. Sentinel records the result as Promoted, Held, or Rejected.

This keeps large indicator volumes in Sentinel for analysis while only high-confidence indicators are pushed into MDO for enforcement.

</details>

---

## 4. Indicator Expiration and Refresh

<details>
<summary><strong>TTL Management and Re-Detection Handling</strong></summary>    

### Expiration TTL by Confidence Level Examples

| Level | TTL | No-Activity Trigger |
|-------|-----|-------------------|
| High (Risk >= 85) | 180 days | Mark for review after 30 days inactive |       
| Medium (Risk 75–84) | 90 days | Auto-expire after 14 days inactive |
| Low (Risk < 75) | 30 days | Auto-expire after 7 days inactive |

### Refresh Strategy

If indicator is re-detected after expiration (or dormancy):
1. Re-ingestion: New ingestion timestamp
2. New correlation: New detection events trigger re-evaluation
3. Re-promotion: Criteria evaluated again; promoted if threshold met
4. TTL Reset: New expiration window begins

</details>

---

## 5. Automation Workflows

<details>
<summary><strong>Automate Promotion and Expiration</strong></summary>      

### Promotion Logic App (Triggered by Sentinel Alert)

1. Query promotion candidates from Sentinel
2. Evaluate criteria (RiskScore >= 75, Detections >= 2)
3. Check TABL capacity (pause if > 95%)
4. Call Graph API to add to TABL
5. Update Sentinel status "promoted"
6. Log audit event

### Expiration Logic App (Weekly Schedule)

1. Query dormant indicators (no activity > TTL days)
2. Check each against TABL to confirm still enforced
3. For each dormant: remove via Graph API DELETE
4. Update Sentinel status "expired"
5. Audit logging for compliance
6. Summary report email

</details>

---

## 6. Monitoring & Alerts

<details>
<summary><strong>Lifecycle Dashboards and KQL Queries</strong></summary>        

### Indicator Status Summary

\\\kusto
CustomIndicator_CL
| summarize
    Ingested = countif(IndicatorStatus == "ingested"),
    Promoted = countif(IndicatorStatus == "promoted"),
    Expired = countif(IndicatorStatus == "expired")
\\\

### TABL Occupancy

\\\kusto
CustomIndicator_CL
| where IndicatorStatus == "promoted"
| summarize Count = count() by IndicatorType
| extend Occupancy_Pct = (Count * 100.0) / 15000
\\\

### Alerts

- **TABL > 90% capacity:** Pause promotions; escalate to architecture team      
- **Promotion failures > 5/hour:** Investigate Graph API access
- **Zero detections 7 days post-promotion:** Review analytic rule quality

</details>

---

## 7. Necessary Components

<details>

### Foundation & Infrastructure
- Service Principals / Entra apps: Ingestion (Sentinel Upload API with workspace-scoped Sentinel Contributor), Promotion (MDO enforcement API path), Expiration

### Data Ingestion
- Logic App: Internal indicator source ingestion
- Sentinel Upload API integration for STIX objects (Logic App/custom app/TIP using `stixobjects` payload)
- Deduplication logic (identify duplicate indicators)

### Sentinel Analytics
- EMail template analytic rules 

### Promotion & Lifecycle Automation
- **Logic App: Promote-Indicator-to-TABL** (core promotion with capacity checks, dedup, throttle retry)
- **Logic App: Expire-Stale-Indicators** (weekly scheduled demotion of dormant indicators)

### Monitoring & Operations
- 5 critical alerts (capacity breach, promotion failures, API auth issues, feed down, zero detections)
- Action groups for SOC escalation (email, SMS, incident platform)

---

## 8. Top Effort Items

Based on the project complexity, these three items will consume the majority of implementation effort:

1. **Promotion Logic App Development & Error Handling** 

**What's involved:**
- Logic App design: Trigger → query → validate → capacity check → dedup check → Graph API call → audit log
- Error paths: Retry logic for 429/500 errors, separate handling for 403/permission issues, exception queue for manual review
- TABL capacity modeling: Track occupancy by type, reserve headroom, implement promotion pause/resume

---

### 2. **Production Hardening, Monitoring, and 14-Day Validation Checkpoint** (~140–180 hours)

**Why it's effort-intensive:**
- Requires planning, deploying, and monitoring infrastructure changes in production with high availability requirements
- Full rollout is staged (domains → URLs → hashes → spoofed senders) to minimize blast radius, but each stage requires monitoring & approval
- Operational readiness is critical: dashboards, alerts, runbooks, SOC training must all work together
- First 14 days post-production are intensive: daily engineering + SOC reviews, runaway promotion detection, exception handling validation

**What's involved:**
- Production infrastructure setup: Service principal rotation, separate prod/non-prod identities, logging workspace isolation
- Monitoring & alerting deployment: 5+ critical alerts, dashboards, KPI tracking
- Phased rollout execution: Week 1 domains (15K capacity), Week 2 URLs (15K), Week 3 hashes (15K), Week 4 spoofed senders (1K)
- 14-day validation checkpoint: Daily reviews, performance trending, false-positive rate tracking, capacity consumption review
- SOC training & on-call rotation: Handoff materials, runbooks, escalation procedures

**Mitigation:**
- Start infrastructure prep in Week 13 (before pilot sign-off)
- Plan phased rollout stages in Weeks 12–13; identify success criteria for each stage gate
- Assign senior architect (0.2 FTE) + SOC lead (0.8 FTE) + on-call rotation starting Week 13

---

## 9. Resources

- **Sentinel STIX Objects Upload API:** https://learn.microsoft.com/en-us/azure/sentinel/stix-objects-api
- **Connect TI platform with Upload API:** https://learn.microsoft.com/en-us/azure/sentinel/connect-threat-intelligence-upload-api
- **Threat intelligence in Sentinel (connectors and deprecation guidance):** https://learn.microsoft.com/en-us/azure/sentinel/understand-threat-intelligence
- **TABL Management:** https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-manage

---
