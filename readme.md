# MS-ISR Log Source Validation Workbook

An Azure Sentinel workbook that validates diagnostic log ingestion across Azure Government workloads for **Authority to Operate (ATO)** preparation. It maps **70 log sources** across **22 Azure services** to **NIST SP 800-53 Rev 4** control families and provides a real-time audit view of what's flowing, what's stale, and what's missing.

---

## Problem

ATO assessments require evidence that audit logging is enabled and operational across all in-scope systems. In Azure Government environments with dozens of workloads, manually verifying diagnostic settings and log ingestion is tedious, error-prone, and hard to repeat. Assessors need to see — at a glance — which log sources satisfy AU-2, AU-3, AU-12, and related controls.

## What This Workbook Does

The workbook queries your Log Analytics workspace and compares what **should** be ingesting (based on the MS-ISR baseline) against what **actually** is. For each of the 70 expected log sources, it determines:

| Status | Audit Result | What It Means |
|--------|-------------|---------------|
| **Ingesting** | Pass | Data is actively flowing into the workspace |
| **Configured** | Review | The table schema exists but no records were found in the selected time range — verify diagnostic settings are still active |
| **Not Configured** | Finding | The table does not exist in the workspace schema — diagnostic settings need to be enabled |

### Features

- **Coverage bar chart** — Stacked bar chart showing Ingesting (green), Configured (yellow), and Not Configured (red) counts per workload
- **Filterable parameters** — Subscription, Workspace, Time Range, Workload, NIST Control Family, Table, Status, and Audit Result
- **Hierarchical table** — Rows grouped by workload, expandable to individual log sources with status icons, last record timestamp, NIST control mappings, and direct links to Microsoft Learn documentation
- **Sample data panel** — Click any row to load the 50 most recent records from that table, with automatic actor/identity resolution (UPN, service principal, managed identity, or GUID → display name lookup)
- **Excel export** — Export the full table for offline evidence documentation

### Azure Workloads Covered

Entra ID, Azure Activity, Automation Account, Container Registry, Data Transfer, AKS, API Management, AVD (Host Pool, AppGroup, Workspace), Application Gateway, Azure Firewall, Key Vault, Load Balancer, Log Analytics / Sentinel, Network Interface, NSG, Public IP (DDoS), Recovery Services Vault (Backup + Site Recovery), Cognitive Search, Virtual Network Gateway, and VNET Flow Logs.

### NIST 800-53 Control Families

Each log source is mapped to one or more control families:

| Family | Description |
|--------|-------------|
| **AC** | Access Control |
| **AU** | Audit & Accountability |
| **CM** | Configuration Management |
| **CP** | Contingency Planning |
| **IA** | Identification & Authentication |
| **IR** | Incident Response |
| **RA** | Risk Assessment |
| **SC** | System & Communications Protection |
| **SI** | System & Information Integrity |

---

## Repository Contents

| File | Purpose |
|------|---------|
| `msisr.json` | The Sentinel workbook definition (JSON). This is what gets deployed to Azure. |
| `deploy_workbook.ps1` | PowerShell script to deploy the workbook via `Invoke-AzRestMethod` to Azure Government |
| `generate_workbook.py` | Python script that can regenerate the workbook JSON from structured inputs |
| `component_nist_mapping.csv` | CSV mapping of Azure workloads → log sources → NIST controls |
| `grounding/` | Reference documents: MS-ISR default logging baseline and NIST 800-53 control descriptions |
| `LOG_VALIDATION_TRACKER.md` | Tracking document for log source validation progress |

## Deployment

### Prerequisites

- Azure PowerShell module (`Az`) authenticated to your Azure Government tenant
- Contributor or Workbook Contributor role on the target resource group
- A Log Analytics workspace with Microsoft Sentinel enabled

### Deploy

1. Update `deploy_workbook.ps1` with your subscription ID, resource group, workspace resource ID, and workbook GUID
2. Run:

```powershell
Connect-AzAccount -Environment AzureUSGovernment
.\deploy_workbook.ps1
```

The workbook will appear under **Microsoft Sentinel → Workbooks → My workbooks**.

## How to Use

1. Open the workbook in Sentinel
2. Select your **Subscription** and **Log Analytics Workspace**
3. Set the **Time Range** — 14 days recommended to avoid false negatives on low-volume tables
4. Review the **Coverage by Workload** bar chart for a high-level view
5. Expand the **Table Ingestion Status** section and drill into individual workloads
6. Filter by **Audit Result** = `Review` + `Finding` to see everything that needs attention
7. Click any row to view **Sample Data** and verify the content of ingested records
8. **Export to Excel** for your ATO evidence package

## Important Disclaimer

This workbook is an **operational aid** for validating log ingestion status. It does **not** replace your security assessment, System Security Plan (SSP), or formal ATO package requirements. Use the results to identify gaps, then work with your assessor to document control implementations appropriately.

---

## Architecture Notes

The workbook uses two detection methods for each log source:

1. **Schema check** — Uses `getschema` to determine if the table exists in the workspace (handles both resource-specific and Azure Diagnostics modes)
2. **Data check** — Queries for `max(TimeGenerated)` within the selected time range to confirm active ingestion

For tables that can arrive via either resource-specific tables or `AzureDiagnostics` (e.g., AVD, Application Gateway, Azure Firewall, Key Vault, Recovery Services), both paths are checked. The workbook resolves whichever mode is active.

The sample data panel includes GUID-to-display-name resolution by joining against `SigninLogs`, `AADServicePrincipalSignInLogs`, and `AADManagedIdentitySignInLogs` to translate opaque object IDs into readable UPNs or service principal names.

## License

Internal use. Not intended for public distribution.
