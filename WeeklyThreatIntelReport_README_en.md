# Weekly Threat Intelligence Report

> A Microsoft Security Copilot Standard Agent solution that uses Microsoft Threat Intelligence and Microsoft Defender to collect threat intelligence released or updated in the past week, filter it by industry, assess whether CVE-related threats affect your environment, and email a CISO-ready HTML report.

## Overview

`WeeklyThreatIntelReport` is designed for executive-level weekly threat briefings. It automates the following workflow:

- Collects recent threat intelligence from Microsoft Threat Intelligence
- Filters the findings by industry or sector
- Focuses on indicators of compromise, threat actors, campaigns, and vulnerability-based attacks
- Checks whether CVE-related threats impact assets in your Defender environment
- Produces an HTML email report tailored for a CISO audience

This solution is intended to answer a simple but important question:

**Which newly released threats matter to our industry, and which of them actually matter to our organization?**

## Architecture

```
┌──────────────────────────────┐
│ Microsoft Security Copilot   │
│ Standard Agent               │
│ WeeklyThreatIntelReport      │
└──────────────┬───────────────┘
               │
               ├──────────────▶ ThreatIntelligence.DTI
               │                - Threat articles
               │                - Industry-filtered intelligence
               │                - CVE summaries
               │
               ├──────────────▶ Defender KQL
               │                - Impacted devices
               │                - Affected software
               │                - Exposure context
               │
               └──────────────▶ Azure Logic App
                                - HTML email delivery
```

## Included Files

| File | Description |
|------|-------------|
| `WeeklyThreatIntelReport.yaml` | Security Copilot Agent manifest |
| `WeeklyThreatIntelReport_card.html` | Visual plugin card summary |
| `WeeklyThreatIntelReport_LogicApp_ARM.json` | ARM template for the Logic App used to send HTML email |
| `WeeklyThreatIntelReport_README.md` | Japanese README |
| `WeeklyThreatIntelReport_README_en.md` | English README |

## Prerequisites

- Access to **Microsoft Security Copilot**
- Access to **Microsoft Defender XDR / Defender for Endpoint Advanced Hunting**
- Availability of the **ThreatIntelligence.DTI** skillset
- An **Azure subscription** where you can deploy a Logic App
- A valid **Office 365 Outlook connector** in Logic Apps for outbound email

## Setup

### 1. Deploy the Logic App

Deploy the Logic App used for HTML email delivery:

```powershell
az deployment group create `
  --resource-group <resource-group-name> `
  --template-file WeeklyThreatIntelReport_LogicApp_ARM.json `
  --parameters emailAddress="ciso@example.com"
```

After deployment, open the `office365` API connection in Azure Portal and complete the authentication step if required.

### 2. Upload the Security Copilot Agent

1. Open the Security Copilot portal
2. Go to `Settings` → `Custom plugins` → `Add plugin`
3. Upload `WeeklyThreatIntelReport.yaml`
4. Enable the plugin

### 3. Provide Setup Parameters

When configuring the agent, enter the following values:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `LogicAppSubscriptionId` | Azure subscription ID where the Logic App is deployed | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `LogicAppResourceGroup` | Resource group name containing the Logic App | `rg-securitycopilot` |
| `LogicAppWorkflowName` | Logic App workflow name | `PluginLogicApp_WeeklyThreatIntelReport` |
| `Industry` | Industry or sector used to filter threat intelligence | `Financial` |

## Agent Design

### Skills Used

| Skill | Type | Purpose |
|-------|------|---------|
| `GenerateThreatIntelReport` | Agent | Main orchestration skill |
| `FindThreatIntelligence` | ThreatIntelligence.DTI | Retrieves recent threat intelligence articles and intelligence findings |
| `GetCvesByIdsDti` | ThreatIntelligence.DTI | Retrieves CVE summaries and impacted technology details |
| `GetAssetsImpactedByCve` | KQL (Defender) | Finds impacted devices and software in your environment for a given CVE |
| `SendThreatReportEmail` | LogicApp | Sends the generated HTML report by email |

### Workflow

```
1. Collect threat intelligence released or updated in the past 7 days
2. Filter and prioritize findings relevant to the chosen industry
3. Extract CVEs mentioned in the threat intelligence
4. Query Defender to identify impacted assets for those CVEs
5. Generate a CISO-focused summary and recommendations
6. Build an HTML email report
7. Send the report through Logic App
```

## Report Content

The report is optimized for executive-level consumption and includes:

- **Executive Summary**
  - Number of major threats this week
  - Number of critical threats
  - Number of CVEs affecting your environment
  - Three short business-facing takeaways

- **Latest Threat Overview Table**
  - Threat / actor / campaign name
  - Related CVEs
  - Internal impacted assets
  - Short description of why it matters

- **Highlighted Critical Threats**
  - Industry relevance
  - Evidence of active exploitation
  - Internal exposure
  - Recommended actions

- **Watchlist Threats**
  - Threats that may not currently have major internal impact but should remain under observation

- **CISO Recommendations**
  - Top three priorities for the week
  - Risks to escalate to executive leadership
  - Required follow-up actions for SOC, IT operations, and vulnerability management teams

## Logic App Input Schema

The `SendThreatReportEmail` skill sends the following JSON payload to the Logic App:

```json
{
  "ReportHtml": "<html>...full generated HTML report...</html>"
}
```

## Defender KQL Role

`GetAssetsImpactedByCve` is used to validate whether the threats surfaced by Microsoft Threat Intelligence actually affect your environment.

The query returns:

- Device names
- Software vendor / name / version
- Recommended security update
- CVSS score
- Exploit availability
- Exposure level
- Internet-facing status
- Sensor health
- Onboarding state

This bridges the gap between external threat intelligence and internal exposure validation.

## Typical Use Cases

- Weekly threat briefings for a CISO or security leadership team
- Executive cyber risk updates tied to industry-specific threats
- Prioritization of vulnerability response based on real threat intelligence
- A concise bridge between threat intelligence and asset exposure

## Troubleshooting

| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| No email is received | Logic App Office 365 connection is not authenticated | Re-authenticate the API connection in Azure Portal |
| Logic App is not triggered | Incorrect SubscriptionId / ResourceGroup / WorkflowName | Recheck the agent setup values |
| No impacted assets are returned | The environment may not contain the affected software | Treat as a valid result and report as no current internal impact |
| Too few threat results | The industry filter may be too narrow, or there were few relevant releases in the last 7 days | Test with a broader industry term |
| Descriptions are still in English | DTI results may be returned in English | Confirm the agent instructions are summarizing results in the desired language |

## Customization Ideas

- Expand the reporting window from 7 days to 30 days
- Support multiple industries in a single run
- Add a more detailed SOC-oriented appendix
- Deliver the report to Teams, SharePoint, or another workflow target instead of email

## License

MIT