# MDO GCC High — User-Reported Submission Automation (Logic App)

Companion to [MDO-GCCH-FP-FN-Guidance.md](MDO-GCCH-FP-FN-Guidance.md). This document describes a Logic App pattern that takes every message landing in the internal user-reporting mailbox and turns it into a fully assembled "case packet" — pre-populated with the exact fields Microsoft Support will ask for — and fans it out to three notification channels in parallel: an ITSM ticket, a Teams channel post, and an email to tenant admins.

The pattern is built for GCC High constraints. There is no public Microsoft 365 / Defender Support API, so the workflow stops at "human ready to click *New service request*." Every other step is automated.

## Architecture

```
Outlook (shared reporting mailbox)
        │  new mail trigger
        ▼
Logic App (Standard, Azure Gov)
        │
        ├── HTTP → Azure Function: Parse-EML
        │           ▸ MIME parse, extract headers
        │           ▸ X-MS-Exchange-Organization-Network-Message-Id
        │           ▸ Internet-Message-ID, From/To/Subject/Date
        │           ▸ URL extraction + redirect chain (best effort)
        │           ▸ SHA-256 hash per attachment
        │
        ├── Compose → Case packet (markdown)
        │
        └── Parallel branches
             ├── HTTP POST → ServiceNow /api/now/table/incident
             ├── Teams connector → channel post (adaptive card)
             └── Outlook connector → email tenant admin DL
```

## Prerequisites

- Azure Government subscription with Logic Apps Standard (or Consumption) and Functions enabled.
- Shared reporting mailbox in the GCCH tenant (the one referenced under MDO → User reported settings → *"Microsoft and my reporting mailbox"* → mailbox-only option).
- Service principal / managed identity for the Logic App with:
  - `Mail.Read` on the reporting mailbox (Exchange Online application access policy scoped to that single mailbox).
  - `SecurityEvents.Read.All` if you want to enrich with Defender alerts.
- ServiceNow instance with a basic-auth or OAuth API account and `incident.write` table access (skip if not using ITSM branch).
- Teams channel with an incoming webhook OR a Teams connector-enabled service account.
- Tenant admin distribution list address for the email branch.
- Azure Function App (PowerShell or Node) in the same Azure Gov region as the Logic App.

## What the workflow does

1. Triggers on every new mail in the reporting mailbox.
2. Validates the message has at least one `.eml` or `.msg` attachment; otherwise stops.
3. Sends the attachment to `Parse-EML` (Azure Function) which returns:
   - `networkMessageId`, `internetMessageId`, `dateUtc`, `fromAddress`, `toAddresses`, `subject`
   - `urls[]` (deduped, lowercased)
   - `attachments[]` with `filename`, `contentType`, `sha256`, `sizeBytes`
4. Builds a markdown case packet matching the checklist in the guidance doc (NMID, Internet Message ID, sender, recipients, current verdict placeholder, hashes, TABL recommendation).
5. Fans out to three branches concurrently. Failure of one branch does not stop the others.

## Workflow JSON

A complete Consumption Logic App workflow definition is in [logic-app-mdo-submission.json](logic-app-mdo-submission.json). Deploy with:

```powershell
az login --service-principal -u <appId> -p <secret> --tenant <tenantId>
az cloud set --name AzureUSGovernment
az group create -n rg-mdo-automation -l usgovvirginia
az deployment group create -g rg-mdo-automation `
  --template-file logic-app-mdo-submission.json `
  --parameters logicAppName=la-mdo-submission `
               functionAppUrl=https://func-mdo-parser.azurewebsites.us/api/Parse-EML `
               functionKey=<function-key> `
               serviceNowInstance=mytenant.service-now.us `
               serviceNowAuth=<basic-auth-b64> `
               teamsWebhookUrl=https://outlook.office365.us/webhook/... `
               adminEmail=secops-admins@contoso.us
```

## Azure Function: Parse-EML (PowerShell)

```powershell
using namespace System.Net
param($Request, $TriggerMetadata)

Add-Type -AssemblyName System.Net.Mail
$bytes = [Convert]::FromBase64String($Request.Body.eml)
$tmp   = [IO.Path]::GetTempFileName() + '.eml'
[IO.File]::WriteAllBytes($tmp, $bytes)

# Parse MIME with MimeKit (install via requirements.psd1)
Import-Module MimeKit
$msg = [MimeKit.MimeMessage]::Load($tmp)

$headers = @{}
foreach ($h in $msg.Headers) { $headers[$h.Field] = $h.Value }

$urls = @()
$body = $msg.HtmlBody + "`n" + $msg.TextBody
$urlRx = [regex]'https?://[^\s"''<>]+'
foreach ($m in $urlRx.Matches($body)) { $urls += $m.Value.ToLowerInvariant() }
$urls = $urls | Sort-Object -Unique

$attachments = @()
foreach ($a in $msg.Attachments) {
    $ms = New-Object IO.MemoryStream
    $a.Content.DecodeTo($ms)
    $data = $ms.ToArray()
    $sha = [BitConverter]::ToString(
        [Security.Cryptography.SHA256]::Create().ComputeHash($data)
    ).Replace('-', '').ToLowerInvariant()
    $attachments += @{
        filename    = $a.FileName
        contentType = $a.ContentType.MimeType
        sha256      = $sha
        sizeBytes   = $data.Length
    }
}

$out = @{
    networkMessageId  = $headers['X-MS-Exchange-Organization-Network-Message-Id']
    internetMessageId = $msg.MessageId
    dateUtc           = $msg.Date.UtcDateTime.ToString('o')
    fromAddress       = ($msg.From | ForEach-Object { $_.ToString() }) -join ', '
    toAddresses       = ($msg.To   | ForEach-Object { $_.ToString() }) -join ', '
    subject           = $msg.Subject
    urls              = $urls
    attachments       = $attachments
}

Remove-Item $tmp -Force -ErrorAction SilentlyContinue
Push-OutputBinding -Name Response -Value ([HttpResponseContext]@{
    StatusCode = [HttpStatusCode]::OK
    Body       = ($out | ConvertTo-Json -Depth 6)
    Headers    = @{ 'Content-Type' = 'application/json' }
})
```

`requirements.psd1`:

```powershell
@{ 'MimeKit' = '4.*' }
```

## Operational notes

- The Logic App never opens the support case itself. The Teams / email / ITSM payload includes a direct deep-link to `https://portal.office365.us/AdminPortal/Home#/support/cases/create` and the full packet ready to paste.
- The ITSM branch creates an Incident with `short_description = "MDO user-reported: <subject>"` and the packet in `description`. Adjust `assignment_group` and `category` to your ITSM taxonomy.
- The Teams branch posts an adaptive card with three buttons: *Open GCCH support portal*, *Copy packet*, *Mark resolved*.
- The email branch sends from the Logic App's managed identity via Graph `sendMail`. If you prefer Exchange Online connector, switch the `Send_an_email_(V2)` action.
- Add a TABL pre-staging step (commented in the JSON) by calling an Azure Function that runs `New-TenantAllowBlockListItems` via Exchange Online PowerShell with a certificate-based service principal. Keep this off by default; turning it on means provisional blocks land before a human has triaged.

## Cost shape

- Consumption Logic App: ~$0.000025 per action, ~50 actions per message = $0.00125 / report.
- Function: under 1 second per parse, well within the free grant for typical volumes.
- ServiceNow / Teams / Outlook calls billed per their own SKUs (no Logic App connector premium tier needed for the built-ins used here).

## What this does *not* do

- It does not file the Microsoft Support case. That step is GCCH-portal only.
- It does not pull a Microsoft verdict. The Defender portal will still return *"Further investigation needed."*
- It does not bypass any data boundary control. The Function runs inside Azure Government in your tenant; nothing leaves the boundary.
