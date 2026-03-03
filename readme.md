# ATO Log Source Validation Tracker

> **Purpose:** Map every Azure resource type in the landing zone to its diagnostic categories, expected Log Analytics tables, required diagnostic mode, and AzureDiagnostics fallback. Validate each against live workspace data.  
> **Workbook:** `generate_workbook.py` → `msisr.json`  
> **Team Reference:** Share this doc so everyone knows what's checked, what's missing, and what the customer needs to fix.  
> **Wiki Source:** Azure DevOps — MSISR / Default Logging Configuration  
> **Last Updated:** 2026-03-02  
> **Total Workbook Entries:** 70 across 22 workloads

---

## How to Verify (Team Instructions)

### Quick Verification Steps

1. **Open the ATO workbook** in Sentinel → Workbooks → "ATO"
2. **Select a workload** from the dropdown filter (ordered to match wiki)
3. **Check each table's status:**
   - **Ingesting** = data flowing, last record date shown ✅
   - **Configured - No Activity** = table exists in schema but 0 records in time range ⚠️
   - **Not Configured** = table does not exist in workspace ❌
4. **Compare against this tracker** — each workload section below shows expected tables and verified counts

### Verify via KQL (Log Analytics)

```kql
// Set your workspace ID
let wsId = "5605c9c4-0f2a-49f5-ae19-d7f8af033df7";

// Check a specific table (replace TableName)
TableName
| summarize Count=count(), Last=max(TimeGenerated)

// Check AzureDiagnostics for a resource type
AzureDiagnostics
| where ResourceType == 'RESOURCE_TYPE_HERE'
| summarize count() by Category
```

### Verify via PowerShell

```powershell
$wsId = "5605c9c4-0f2a-49f5-ae19-d7f8af033df7"
$q = "TableName | summarize Count=count(), Last=max(TimeGenerated)"
(Invoke-AzOperationalInsightsQuery -WorkspaceId $wsId -Query $q).Results
```

### Rebuild & Deploy Workflow

```powershell
cd C:\tools\nistframework\ato
python generate_workbook.py     # Generates msisr.json
.\deploy_workbook.ps1           # Deploys to Sentinel
```

### What Each Status Means for ATO

| Workbook Status | ATO Impact | Action Required |
|-----------------|-----------|-----------------|
| Ingesting | ✅ Pass | None — logs flowing |
| Configured - No Activity | ⚠️ Pass (conditional) | Verify resource exists and is active; may need longer time range |
| Not Configured | ❌ Finding | Customer must enable diagnostic settings for this resource type |

---

## Key Concepts

### Diagnostic Destination Modes

| Mode | Where data lands | When to use |
|------|-----------------|-------------|
| **Resource specific** | Dedicated tables (e.g., `CoreAzureBackup`, `AZFWApplicationRule`) | Microsoft recommended for most new resources |
| **Azure diagnostics** | `AzureDiagnostics` table (shared, filtered by `ResourceType` + `Category`) | Legacy; still required for some categories that have no resource-specific table |

**Critical:** Some resources support BOTH modes. The workbook must check BOTH locations using `AZDIAG_FALLBACK` in the generator. A single diagnostic setting can only be one mode — some resources need **two separate diagnostic settings** (one per mode).

### Workbook Detection Logic

For each log source the workbook checks:
1. **Resource-specific table** — query the dedicated table directly
2. **AzureDiagnostics fallback** — query `AzureDiagnostics | where ResourceType == 'X' and Category == 'Y'`
3. **Schema check** — `getschema` to detect if table exists even with no data

If either location has data → **Ingesting**  
If table exists but no data in time range → **Validated - No Data**  
If table doesn't exist at all → **Not Configured**

---

### ⚠️ No Data in Last 30 Days — Customer Follow-Up Required

The following workloads have historical data in the workspace but **zero records in the last 30 days**. Extend the workbook time range to verify historical data exists, then confirm with the customer whether the resources are still active and diagnostic settings are still configured.

| Workload | Last Record | Total Historical Records | Notes |
|----------|-------------|------------------------|-------|
| Azure Data Transfer | 2024-07-04 | 45,180 | `DataTransferOperations` — resource may be inactive |
| Azure Kubernetes Service | 2025-12-09 | 122,890,486 | `AzureDiagnostics` (MANAGEDCLUSTERS) — 21 clusters, stopped ~3 months ago |
| Azure API Management | 2024-03-20 | 2 | `AzureDiagnostics` (SERVICE) — only 2 records ever, GatewayLogs category |

---

## Validation Status Summary

> **Order matches MS-ISR Default Logging Configuration wiki exactly.**

| # | Wiki Abbrev | Workload | Entries | Status | Notes |
|---|-------------|----------|---------|--------|-------|
| 1 | — | [Entra ID](#entra-id) | 12 | ✅ Complete | Tenant-level; 7 ingesting, 5 schema-only |
| 2 | — | [Azure Activity](#azure-activity) | 1 | ✅ Complete | 3 subscriptions, 87K+ records (30d), actively ingesting |
| 3 | AA | [Azure Automation Account](#azure-automation-account) | 1 | ✅ Complete | AzureDiagnostics; 3 categories ingesting (AuditEvent, JobLogs, JobStreams) |
| 4 | ACR | [Azure Container Registry](#azure-container-registry) | 2 | ✅ Complete | Resource-specific; LoginEvents ingesting, RepositoryEvents schema-only |
| 5 | ADT | [Azure Data Transfer](#azure-data-transfer) | 1 | ⚠️ Needs Review | 45K historical records (last 2024-07-04), no recent activity |
| 6 | AKS | [Azure Kubernetes Service](#azure-kubernetes-service) | 1 | ⚠️ Needs Review | 122M historical records (last 2025-12-09), 0 in 30d |
| 7 | APIMgmt | [Azure API Management](#azure-api-management) | 1 | ⚠️ Needs Review | 2 historical records (last 2024-03-20), 0 in 30d |
| 8 | AppGroup | [Azure Virtual Desktop (AppGroup)](#azure-virtual-desktop-appgroup) | 3 | ✅ Complete | Resource-specific WVD tables filtered by applicationgroups; Management ingesting (2,652 records), Checkpoints/Errors schema-only |
| 9 | ApplicationGateway | [Azure Application Gateway](#azure-application-gateway) | 3 | ✅ Complete | Resource-specific tables (AGWAccessLogs, AGWPerformanceLogs, AGWFirewallLogs); all schema-only, 0 data — no AppGW resources deployed |
| 10 | Firewall | [Azure Firewall](#azure-firewall) | 4 | ✅ Complete | Wiki 1-for-1 (4 of 4); AzDiag mode; AppRule/NetRule/DNS ingesting, IDPS not enabled (Premium feature) |
| 11 | HostPool | [Azure Virtual Desktop (HostPool)](#azure-virtual-desktop) | 11 | ✅ Complete | All 11 MS-supported HostPool categories; 7 ingesting, 4 schema-only; WVDFeeds removed (Workspace-only category) |
| 12 | KeyVault | [Azure Key Vault](#azure-key-vault) | 1 | ✅ Complete | Legacy AzureDiagnostics; AuditEvent ingesting from 3 subs |
| 13 | LoadBalancer | [Azure Load Balancer](#azure-load-balancer) | 1 | ✅ Complete | Only 1 MS-supported category (LoadBalancerHealthEvent → ALBHealthEvent); schema exists, 0 data |
| 14 | loganalytics | [Log Analytics Workspace](#log-analytics-workspace) | 5 | ✅ Complete | 3 LA diagnostic categories + 2 Sentinel audit tables; LAQueryLogs ingesting (1,535 in 30d), SentinelHealth/SentinelAudit ingesting, LAJobLogs/LASummaryLogs schema-only |
| 15 | NIC | [Network Interface](#network-interface) | 1 | ✅ Complete | No diagnostic log categories exist — NIC flow visibility comes from NSG diag settings + VNET Flow Logs |
| 16 | NSG | [Network Security Group](#network-security-group) | 2 | ✅ Complete | 2 enabled categories (Event + RuleCounter); 44M all-time; deployed via Azure Policy `setbypolicy` to LA + Event Hub |
| 17 | PublicIP | [Public IP Address](#public-ip-address) | 3 | ✅ Complete | 3 DDoS categories; diag settings enabled via policy but to Event Hub only (NOT LA); no DDoS Protection Plan in tenant |
| 18 | RecoveryVault | [Azure Recovery Services Vault](#azure-recovery-services-vault) | 9 | ✅ Complete | Resource-specific; 6 ingesting, 3 schema-only |
| 19 | SearchServices | [Azure Cognitive Search](#azure-cognitive-search) | 1 | ✅ Complete | AzureDiagnostics; 624 records (30d), OperationLogs, 1 sub |
| 20 | VNetGW | [Virtual Network Gateway](#virtual-network-gateway) | 1 | ✅ Complete | Not Configured — 0 data; ResourceFilter fixed to VIRTUALNETWORKGATEWAYS |
| 21 | Workspace | [Azure Virtual Desktop (Workspace)](#azure-virtual-desktop-workspace) | 4 | ✅ Complete | Resource-specific WVD tables filtered by workspaces; all 4 actively ingesting (Feeds 1,634, Mgmt 1,006, Checkpoints 577, Errors 8 in 30d) |
| 22 | — | [VNET Flow Logs](#vnet-flow-logs) | 2 | ✅ Complete | Both tables ingesting: NTANetAnalytics 2.9M + AzureNetworkAnalytics_CL 4.5M all-time; 20+ Network Watcher flow configs active |

### MS-ISR Wiki Cross-Reference Checklist

Source: Azure DevOps Wiki — MSISR / Default Logging Configuration

**Entra ID Logs**
- [x] ADFSSignInLogs → `ADFSSignInLogs` ✅
- [x] ManagedIdentitySignInLogs → `AADManagedIdentitySignInLogs` ✅
- [x] MicrosoftGraphActivityLogs → `MicrosoftGraphActivityLogs` ✅
- [x] MicrosoftServicePrincipalSignInLogs → `AADServicePrincipalSignInLogs` (same table, wiki uses category name) ✅
- [x] ProvisioningLogs → `AADProvisioningLogs` ✅
- [x] RiskyServicePrincipals → `AADRiskyServicePrincipals` ✅
- [x] RiskyUsers → `AADRiskyUsers` ✅
- [x] ServicePrincipalRiskEvents → `AADServicePrincipalRiskEvents` ✅
- [x] ServicePrincipalSignInLogs → `AADServicePrincipalSignInLogs` ✅
- [x] SignInLogs → `SigninLogs` ✅
- [x] UserRiskEvents → `AADUserRiskEvents` ✅
- [x] (Extra) NonInteractiveUserSignInLogs → `AADNonInteractiveUserSignInLogs` (not in wiki, actively ingesting)
- [x] (Extra) AuditLogs → `AuditLogs` (not in wiki, actively ingesting 20K+ records)

**Azure Activity Logs**
- [x] Tenant Root Directory Activity Logs → `AzureActivity` ✅
- [x] Management Group Activity Logs → `AzureActivity` ✅
- [x] Subscription Activity Logs → `AzureActivity` ✅

**Azure Diagnostics Logs** (wiki order)
- [x] AA (Automation Account) → `AzDiag_AUTOMATIONACCOUNTS` ✅
- [x] ACR (Container Registry) → `ContainerRegistryLoginEvents`, `ContainerRegistryRepositoryEvents` ✅
- [x] ADT (Azure Data Transfer) → `DataTransferOperations` ⚠️ (no data in 30d)
- [x] AKS (Kubernetes Service) → `AzDiag_MANAGEDCLUSTERS` ⚠️ (no data in 30d)
- [x] APIMgmt (API Management) → `AzDiag_SERVICE` ⚠️ (no data in 30d)
- [x] AppGroup (AVD Application Group) → `WVDCheckpoints_AppGroup`, `WVDErrors_AppGroup`, `WVDManagement_AppGroup` (3 entries with AZDIAG_FALLBACK → APPLICATIONGROUPS) ✅
- [x] ApplicationGateway → `AGWAccessLogs`, `AGWPerformanceLogs`, `AGWFirewallLogs` (3 resource-specific entries with AZDIAG_FALLBACK → APPLICATIONGATEWAYS) ✅ Schema only, 0 data
- [x] Firewall → AZFWApplicationRule, AZFWNetworkRule, etc. (7 entries, 3 ingesting via AZDIAG_FALLBACK) ✅
- [x] HostPool (AVD Host Pool) → WVD* tables (12 entries including WVDMultiLinkAdd) ✅
- [x] KeyVault → `AzDiag_VAULTS` ✅
- [x] LoadBalancer → `ALBHealthEvent` (changed from AzDiag to resource-specific) ⚠️ Schema only, 0 data
- [x] loganalytics (LA Workspace) → `LAQueryLogs` (changed from AzDiag to resource-specific) ✅
- [x] NIC → `AzDiag_NETWORKINTERFACES` ℹ️ No diagnostic log categories exist for NICs (expected Not Configured)
- [x] NSG → `AzDiag_NSG_Event` + `AzDiag_NSG_RuleCounter` ✅ 44M records all-time, 2 categories, deployed via Azure Policy
- [x] PublicIP → `AzDiag_PIP_DDoSNotify` + `AzDiag_PIP_DDoSFlow` + `AzDiag_PIP_DDoSReport` ⚠️ Diag settings → Event Hub only, NOT LA; no DDoS Protection Plan
- [x] RecoveryVault → CoreAzureBackup, AddonAzureBackup*, ASR* (9 entries) ✅
- [x] SearchServices → `AzDiag_SEARCHSERVICES` ✅ 624 records (30d)
- [x] VNetGW → `AzDiag_VIRTUALNETWORKGATEWAYS` (ResourceFilter fixed from VPNGATEWAYS) ❌ Not Configured
- [x] Workspace (AVD Workspace) → `WVDFeeds_Workspace`, `WVDManagement_Workspace`, `WVDCheckpoints_Workspace`, `WVDErrors_Workspace` (4 entries with AZDIAG_FALLBACK → WORKSPACES) ✅

**Network Flow Logs**
- [x] VNET Flow Logs → `NTANetAnalytics` (551K) + `AzureNetworkAnalytics_CL` (182K) ✅

---

## AllMetrics Tracking

> **Note:** AllMetrics → `AzureMetrics` table (shared platform table). Not tracked in the ATO log workbook — metrics are performance/health data, not audit logs. This section records which workloads have AllMetrics enabled in their diagnostic settings for future reference.

| Workload | AllMetrics Enabled | Notes |
|----------|-------------------|-------|
| Azure Container Registry | ✅ Yes | `acrc3elz7sharedugva1` — enabled alongside log categories |
| Azure Key Vault | ❌ No | `gateskv001` — logs only (allLogs group), no metrics |

---

## Entra ID

**Service:** Microsoft Entra ID (Azure AD)  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** N/A — tenant-level diagnostic settings  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Entra ID activity logs in Azure Monitor](https://learn.microsoft.com/entra/identity/monitoring-health/howto-integrate-activity-logs-with-azure-monitor-logs) | [ID Protection risk data](https://learn.microsoft.com/entra/id-protection/howto-export-risk-data) | [Graph Activity Logs](https://learn.microsoft.com/graph/microsoft-graph-activity-logs-overview)

### What We Did

1. Queried all 12 Entra ID tables against live workspace
2. All 12 tables have schema present — none are "Not Configured"
3. 7 tables actively ingesting, 5 have schema but no data in 30d range
4. Workbook entries verified — all 12 in `TENANT_TABLES` set, Source shows "Tenant-wide"
5. No changes needed to generator

### Table Status (12 entries in workbook)

| Table | Display Name | NIST Controls | Status | Records (30d) |
|-------|-------------|---------------|--------|---------------|
| `SigninLogs` | Sign-in Logs | AC-2, IA-2, AU-2, AU-3 | ✅ Ingesting | 81,322 |
| `AADNonInteractiveUserSignInLogs` | Non-Interactive Sign-ins | AC-2, IA-2, AU-2, AU-3 | ℹ️ Schema only | 0 |
| `AuditLogs` | Audit Logs | AC-2, AC-6, AU-2, AU-3, AU-12 | ✅ Ingesting | 20,869 |
| `AADServicePrincipalSignInLogs` | Service Principal Sign-ins | AC-2, IA-2, AU-2 | ✅ Ingesting | 1,941,674 |
| `AADManagedIdentitySignInLogs` | Managed Identity Sign-ins | AC-2, IA-2, IA-4, AU-2 | ✅ Ingesting | 6,078,127 |
| `AADProvisioningLogs` | Provisioning Logs | AC-2, AU-2, AU-12 | ℹ️ Schema only | 0 |
| `AADRiskyUsers` | Risky Users | AC-2, IA-5, SI-4, RA-5 | ✅ Ingesting | 47 |
| `AADUserRiskEvents` | User Risk Events | AC-2, IA-5, SI-4, RA-5 | ✅ Ingesting | 58 |
| `AADRiskyServicePrincipals` | Risky Service Principals | AC-2, IA-5, SI-4 | ℹ️ Schema only | 0 |
| `AADServicePrincipalRiskEvents` | SP Risk Events | AC-2, IA-5, SI-4, RA-5 | ℹ️ Schema only | 0 |
| `ADFSSignInLogs` | ADFS Sign-in Logs | AC-2, IA-2, AU-2 | ℹ️ Schema only | 0 |
| `MicrosoftGraphActivityLogs` | Graph Activity Logs | AC-2, AC-6, AU-2, AU-3, SI-4 | ✅ Ingesting | 10,680,457 |

### Notes

- All tables are tenant-level — no subscription extraction, Source shows "Tenant-wide"
- `AADNonInteractiveUserSignInLogs` has schema but 0 records — may indicate users are authenticating via interactive flows only, or data is outside the query time range
- `ADFSSignInLogs` has schema but 0 records — expected if no ADFS federation is configured
- `AADRiskyServicePrincipals` / `AADServicePrincipalRiskEvents` — expected 0 if no SP risk detections have occurred
- `AADProvisioningLogs` — expected 0 if no provisioning connectors are actively syncing

### Validation KQL

```kql
union isfuzzy=true
    (SigninLogs | summarize Count=count(), Last=max(TimeGenerated) | extend Table='SigninLogs'),
    (AADNonInteractiveUserSignInLogs | summarize Count=count(), Last=max(TimeGenerated) | extend Table='AADNonInteractiveUserSignInLogs'),
    (AuditLogs | summarize Count=count(), Last=max(TimeGenerated) | extend Table='AuditLogs'),
    (AADServicePrincipalSignInLogs | summarize Count=count(), Last=max(TimeGenerated) | extend Table='AADServicePrincipalSignInLogs'),
    (AADManagedIdentitySignInLogs | summarize Count=count(), Last=max(TimeGenerated) | extend Table='AADManagedIdentitySignInLogs'),
    (AADProvisioningLogs | summarize Count=count(), Last=max(TimeGenerated) | extend Table='AADProvisioningLogs'),
    (AADRiskyUsers | summarize Count=count(), Last=max(TimeGenerated) | extend Table='AADRiskyUsers'),
    (AADUserRiskEvents | summarize Count=count(), Last=max(TimeGenerated) | extend Table='AADUserRiskEvents'),
    (AADRiskyServicePrincipals | summarize Count=count(), Last=max(TimeGenerated) | extend Table='AADRiskyServicePrincipals'),
    (AADServicePrincipalRiskEvents | summarize Count=count(), Last=max(TimeGenerated) | extend Table='AADServicePrincipalRiskEvents'),
    (ADFSSignInLogs | summarize Count=count(), Last=max(TimeGenerated) | extend Table='ADFSSignInLogs'),
    (MicrosoftGraphActivityLogs | summarize Count=count(), Last=max(TimeGenerated) | extend Table='MicrosoftGraphActivityLogs')
| project Table, Count, Last
| sort by Table asc
```

---

## Azure Activity

**Service:** Azure Activity Log  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** N/A — subscription-level Activity Log export  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Azure Activity Log](https://learn.microsoft.com/azure/azure-monitor/essentials/activity-log)

### What We Did

1. Queried `AzureActivity` table — actively ingesting from 3 subscriptions
2. Workbook entry verified: single `AzureActivity` entry with SubscriptionId extraction
3. No changes needed

### Source Subscriptions (3)

| Subscription ID | 30d Count | Last Record |
|----------------|-----------|-------------|
| `ac95a806-c9d3-49e7-83ee-7f82e88c2bd3` | 69,410 | 2026-03-02 |
| `787e871a-84ba-43be-86bf-86bd1e408a4a` | 14,478 | 2026-03-02 |
| `b30166b8-dd1b-4fa2-9ad7-057614257b06` | 4,038 | 2026-03-02 |

### Notes

- Wiki states Activity Logs collected at Tenant Root, Management Group, and Subscription scopes
- All 3 visible subscriptions actively ingesting
- Coverage across Root/MG/Sub scopes confirmed by presence of multiple subscriptions

---

## Azure Automation Account

**Resource Provider:** `Microsoft.Automation/automationAccounts`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Azure Diagnostics (legacy)  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Automation diagnostic logs](https://learn.microsoft.com/azure/automation/automation-manage-send-joblogs-log-analytics) | [Supported logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-automation-automationaccounts-logs)

### What We Did

1. Queried AzureDiagnostics for `ResourceType == 'AUTOMATIONACCOUNTS'`
2. Found 3 categories actively ingesting from 1 subscription
3. Checked MS docs — 4th category `DscNodeStatus` exists but no DSC configured
4. Single `AzDiag_AUTOMATIONACCOUNTS` entry covers all categories
5. No changes needed

### Category Breakdown

| Category | 30d Count | Status |
|----------|-----------|--------|
| AuditEvent | 990 | ✅ Ingesting |
| JobLogs | 2,973 | ✅ Ingesting |
| JobStreams | 42,815 | ✅ Ingesting |
| DscNodeStatus | 0 | Not configured (no DSC) |

### Source Subscription: `ac95a806-c9d3-49e7-83ee-7f82e88c2bd3`

---

## Azure Container Registry

**Resource Provider:** `Microsoft.ContainerRegistry/registries`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Resource-specific  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Monitor ACR](https://learn.microsoft.com/azure/container-registry/monitor-service) | [Supported logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-containerregistry-registries-logs)

### What We Did

1. Checked MS docs — ACR has 2 categories, both resource-specific tables
2. Queried workspace — customer uses resource-specific mode (no AzureDiagnostics data for REGISTRIES)
3. Replaced single `AzDiag_REGISTRIES` entry with 2 resource-specific entries
4. ACR went from 1 → 2 workbook entries

### Table Status (2 entries in workbook)

| Table | 30d Count | Status |
|-------|-----------|--------|
| `ContainerRegistryLoginEvents` | 2,092 | ✅ Ingesting |
| `ContainerRegistryRepositoryEvents` | 0 | ℹ️ Schema only (no push/pull events) |

---

## Azure Data Transfer

**Resource Provider:** `Microsoft.AzureDataTransfer/connections/flows`  
**Status:** ⚠️ Needs Review — validated 2026-03-02  
**Diagnostic Mode:** Resource-specific  
**Table:** `DataTransferOperations`  
**MS Docs:** [Supported logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-azuredatatransfer-connections-flows-logs) | [Table reference](https://learn.microsoft.com/azure/azure-monitor/reference/tables/datatransferoperations)

### What We Did

1. Checked MS docs — confirmed 1 diagnostic category: `OperationalLogs` → `DataTransferOperations`
2. Queried live workspace — table exists with 45,180 historical records
3. Time range: 2024-03-20 → 2024-07-04 (no recent data, resource likely inactive)
4. Source subscription: `ac95a806-c9d3-49e7-83ee-7f82e88c2bd3`
5. Workbook entry already present — no changes needed
6. **No data in last 30d** — needs cx follow-up

### Validation KQL

```kql
DataTransferOperations
| summarize RecordCount=count(), MinTime=min(TimeGenerated), MaxTime=max(TimeGenerated), Subs=dcount(_SubscriptionId)
```

### Result

| Table | Records | First Seen | Last Seen | Status |
|-------|---------|------------|-----------|--------|
| DataTransferOperations | 45,180 | 2024-03-20 | 2024-07-04 | Configured - No Activity (no data in 30d window) |

---

## Azure Kubernetes Service

**Resource Provider:** `microsoft.containerservice/managedclusters`  
**Status:** ⚠️ Needs Review — validated 2026-03-02  
**Diagnostic Mode:** AzureDiagnostics (legacy)  
**Table:** `AzureDiagnostics` where `ResourceType == 'MANAGEDCLUSTERS'`  
**MS Docs:** [Monitor AKS](https://learn.microsoft.com/azure/aks/monitor-aks) | [AKS resource logs](https://learn.microsoft.com/azure/azure-monitor/reference/tables/microsoft-containerservice_managedclusters)

### What We Did

1. Checked MS docs — AKS supports both AzureDiagnostics and resource-specific modes
2. Resource-specific tables (`AKSAudit`, `AKSAuditAdmin`, `AKSControlPlane`) exist but have 0 data
3. Customer uses AzureDiagnostics mode — 122M records across 21 AKS clusters
4. Wiki lists 7 categories; customer has 11 (4 extras: cloud-controller-manager, 3x CSI controllers)
5. Single workbook entry `AzDiag_MANAGEDCLUSTERS` covers all categories — no changes needed
6. **No data in last 30d** — last record 2025-12-09, needs cx follow-up

### Validation KQL

```kql
AzureDiagnostics
| where ResourceType == 'MANAGEDCLUSTERS'
| summarize Records=count() by Category
| order by Records desc
```

### Result

| Category | Records | In Wiki? |
|----------|---------|----------|
| kube-audit | 63,050,893 | ✅ |
| kube-audit-admin | 32,382,118 | ✅ |
| cloud-controller-manager | 13,801,814 | Extra |
| cluster-autoscaler | 5,948,516 | ✅ |
| kube-apiserver | 4,584,324 | ✅ |
| guard | 2,512,619 | ✅ |
| kube-controller-manager | 390,252 | ✅ |
| kube-scheduler | 218,460 | ✅ |
| csi-azuredisk-controller | 775 | Extra |
| csi-azurefile-controller | 626 | Extra |
| csi-snapshot-controller | 89 | Extra |

---

## Azure API Management

**Resource Provider:** `Microsoft.ApiManagement/service`  
**Status:** ⚠️ Needs Review — validated 2026-03-02  
**Diagnostic Mode:** AzureDiagnostics (legacy)  
**Table:** `AzureDiagnostics` where `ResourceType == 'SERVICE'`  
**MS Docs:** [Supported logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-apimanagement-service-logs) | [Monitor APIM](https://learn.microsoft.com/azure/api-management/monitor-api-management)

### What We Did

1. Checked MS docs — APIM supports 5 categories: GatewayLogs, WebSocketConnectionLogs, DeveloperPortalAuditLogs, GatewayLlmLogs, GatewayMCPLogs
2. Wiki lists 3: Gateway logs, developer portal audit logs, WebSocket connection logs
3. Live workspace: only 2 records, GatewayLogs category, from 2024-03-20
4. Resource-specific tables (`ApiManagementGatewayLogs`, etc.) do not exist
5. **No data in 30d window** — needs cx follow-up to confirm APIM is still deployed/configured
6. Single workbook entry `AzDiag_SERVICE` covers all categories — no changes needed

### Validation KQL

```kql
AzureDiagnostics
| where ResourceType == 'SERVICE'
| summarize RecordCount=count(), Categories=make_set(Category), MinTime=min(TimeGenerated), MaxTime=max(TimeGenerated)
```

### Result

| Category | Records | Time Range | Status |
|----------|---------|------------|--------|
| GatewayLogs | 2 | 2024-03-20 | No current ingestion — extend time range to verify |

---

## Azure Virtual Desktop

> **Note:** Wiki lists AppGroup, HostPool, and Workspace as separate resource types. All three share the same WVD* tables in Log Analytics. HostPool tracks all **11 MS-supported diagnostic categories** for `Microsoft.DesktopVirtualization/hostPools`. AppGroup has 3 dedicated entries filtered by `_ResourceId has 'applicationgroups'`. Workspace has 4 dedicated entries filtered by `_ResourceId has 'workspaces'`.

**Resource Provider:** `Microsoft.DesktopVirtualization/hostPools`  
**Status:** ✅ Complete — validated 2026-03-02  
**Wiki Entry:** `Session host health, connection activity, agent logs` → **11 entries (all MS-supported HostPool categories)**  
**Diagnostic Mode:** Resource-specific  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [AVD diagnostics with Log Analytics](https://learn.microsoft.com/azure/virtual-desktop/diagnostics-log-analytics) | [Supported HostPool logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-desktopvirtualization-hostpools-logs)

### What We Did

1. Queried all 11 HostPool-supported tables — 7 actively ingesting from hostpools, 4 schema-only
2. AzureDiagnostics `ResourceType == 'HOSTPOOLS'` returned 0 records — customer uses resource-specific mode exclusively
3. **Removed `WVDFeeds`** from HostPool — per [MS Learn](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-desktopvirtualization-hostpools-logs), the Feed category is NOT a HostPool diagnostic category; it belongs to Workspaces only. WVDFeeds is already tracked under the "AVD Workspace" workload (12→11 entries)
4. Wiki's 3 descriptions map to groupings of the 11 categories:
   - **Session host health** → WVDAgentHealthStatus, WVDHostRegistrations, WVDSessionHostManagement
   - **Connection activity** → WVDConnections, WVDConnectionNetworkData, WVDConnectionGraphicsDataPreview, WVDMultiLinkAdd
   - **Agent logs** → WVDCheckpoints, WVDErrors, WVDManagement, WVDAutoscaleEvaluationPooled

### Table Status (11 entries in workbook)

| Wiki Grouping | Table | HostPool All-Time Count | Last Record | Status |
|--------------|-------|------------------------|-------------|--------|
| Session host health | `WVDAgentHealthStatus` | 1,285,861 | 2026-03-02 | ✅ Ingesting |
| Session host health | `WVDHostRegistrations` | 177 | 2026-02-18 | ✅ Ingesting (low volume) |
| Session host health | `WVDSessionHostManagement` | 0 | — | ℹ️ Schema only |
| Connection activity | `WVDConnections` | 4,119 | 2026-02-27 | ✅ Ingesting |
| Connection activity | `WVDConnectionNetworkData` | 26,614 | 2025-11-12 | ℹ️ Historical only |
| Connection activity | `WVDConnectionGraphicsDataPreview` | 0 | — | ℹ️ Schema only |
| Connection activity | `WVDMultiLinkAdd` | 0 | — | ℹ️ Schema only |
| Agent logs | `WVDManagement` | 42,777 | 2026-03-02 | ✅ Ingesting |
| Agent logs | `WVDCheckpoints` | 34,855 | 2026-02-27 | ✅ Ingesting |
| Agent logs | `WVDErrors` | 1,257 | 2026-02-26 | ✅ Ingesting |
| Agent logs | `WVDAutoscaleEvaluationPooled` | 0 | — | ℹ️ Schema only (no autoscale configured) |

### Removed from HostPool (moved to Workspace)

| Table | Reason |
|-------|--------|
| `WVDFeeds` | Feed is a Workspace-only diagnostic category per MS Learn; already tracked under "AVD Workspace" workload |

### Source Subscriptions (2)

| Subscription ID |
|----------------|
| `ac95a806-c9d3-49e7-83ee-7f82e88c2bd3` |
| `787e871a-84ba-43be-86bf-86bd1e408a4a` |

---

## Azure Virtual Desktop (AppGroup)

**Resource Provider:** `Microsoft.DesktopVirtualization/applicationGroups`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Resource-specific (shared WVD tables)  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [AVD diagnostics with Log Analytics](https://learn.microsoft.com/azure/virtual-desktop/diagnostics-log-analytics)

### What We Did

1. Wiki lists AppGroup separately from HostPool — investigated whether AppGroup sends data to its own tables
2. AzureDiagnostics `ResourceType == 'APPLICATIONGROUPS'` → 0 records (resource-specific mode, no legacy data)
3. Checked all WVD tables filtered by `_ResourceId has 'applicationgroups'`:
   - **WVDManagement**: 2,652 records (89 in 30d), last ingested 2026-02-18 ✅
   - **WVDCheckpoints**: 0 records from applicationgroups resources
   - **WVDErrors**: 0 records from applicationgroups resources
4. Added 3 dedicated workbook entries with unique CheckIds (`WVDCheckpoints_AppGroup`, `WVDErrors_AppGroup`, `WVDManagement_AppGroup`) pointing to the shared WVD tables
5. Added 3 AZDIAG_FALLBACK entries mapping to `APPLICATIONGROUPS` resource type (Checkpoint, Error, Management categories)
6. Queries verified against live workspace — all return successfully

### Table Status (3 entries in workbook)

| Table | CheckId | All-Time Count | 30d Count | Last Record | Status |
|-------|---------|---------------|-----------|-------------|--------|
| `WVDManagement` | WVDManagement_AppGroup | 2,652 | 89 | 2026-02-18 | ✅ Ingesting |
| `WVDCheckpoints` | WVDCheckpoints_AppGroup | 0 | 0 | — | ℹ️ Schema only (from applicationgroups) |
| `WVDErrors` | WVDErrors_AppGroup | 0 | 0 | — | ℹ️ Schema only (from applicationgroups) |

### Key Insight

AppGroup resources primarily generate management events (application group creation, updates, user assignments). Checkpoints and Errors from applicationgroups resources are rare. The WVDManagement table is actively ingesting AppGroup-sourced records.

---

## Azure Virtual Desktop (Workspace)

**Resource Provider:** `Microsoft.DesktopVirtualization/workspaces`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Resource-specific (shared WVD tables)  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [AVD diagnostics with Log Analytics](https://learn.microsoft.com/azure/virtual-desktop/diagnostics-log-analytics)

### What We Did

1. Wiki lists Workspace separately from HostPool — investigated which WVD tables receive data from workspace resources
2. AzureDiagnostics `ResourceType == 'WORKSPACES'` (with `ResourceProvider == 'MICROSOFT.DESKTOPVIRTUALIZATION'`) → 0 records (resource-specific mode)
3. Checked WVD tables filtered by `_ResourceId has 'workspaces'` — all 4 are actively ingesting
4. Added 4 dedicated workbook entries with unique CheckIds (`WVDFeeds_Workspace`, `WVDManagement_Workspace`, `WVDCheckpoints_Workspace`, `WVDErrors_Workspace`)
5. Added 4 AZDIAG_FALLBACK entries mapping to `WORKSPACES` resource type (Feed, Management, Checkpoint, Error categories)
6. Queries verified against live workspace — all return successfully

### Table Status (4 entries in workbook)

| Table | CheckId | All-Time Count | 30d Count | Last Record | Status |
|-------|---------|---------------|-----------|-------------|--------|
| `WVDFeeds` | WVDFeeds_Workspace | 32,882 | 1,634 | 2026-03-02 | ✅ Ingesting |
| `WVDManagement` | WVDManagement_Workspace | 32,388 | 1,006 | 2026-03-02 | ✅ Ingesting |
| `WVDCheckpoints` | WVDCheckpoints_Workspace | 29,465 | 577 | 2026-02-27 | ✅ Ingesting |
| `WVDErrors` | WVDErrors_Workspace | 877 | 8 | 2026-02-26 | ✅ Ingesting |

### Key Insight

AVD Workspace resources generate the most WVD data of the three resource types (HostPool, AppGroup, Workspace). WVDFeeds is primarily a workspace-sourced table. All 4 tables are actively ingesting from workspace resources.

---

## Azure Application Gateway

**Resource Provider:** `Microsoft.Network/applicationGateways`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Azure Diagnostics (legacy) / Resource-specific  
**MS Docs:** [Diagnostic logs for Application Gateway](https://learn.microsoft.com/azure/application-gateway/application-gateway-diagnostics) | [Supported logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-network-applicationgateways-logs)

### What We Did

1. Checked MS docs — AppGW has 3 categories: ApplicationGatewayAccessLog, ApplicationGatewayFirewallLog, ApplicationGatewayPerformanceLog
2. Resource-specific tables: `AGWAccessLogs`, `AGWPerformanceLogs`, `AGWFirewallLogs` — all exist in schema, 0 data
3. AzureDiagnostics `ResourceType == 'APPLICATIONGATEWAYS'` — 0 records (all time)
4. **No Application Gateway resources appear to be deployed or configured**
5. Updated workbook from single `AzDiag_APPLICATIONGATEWAYS` entry to 3 resource-specific entries with AZDIAG_FALLBACK
6. All 3 entries will show "Configured - No Activity" (schema exists but no data)

### Category → Table Mapping

| Diagnostic Category | Legacy Table | Resource-Specific Table | Status |
|---------------------|-------------|------------------------|--------|
| ApplicationGatewayAccessLog | AzureDiagnostics (APPLICATIONGATEWAYS) | AGWAccessLogs | ❌ Not Configured |
| ApplicationGatewayFirewallLog | AzureDiagnostics (APPLICATIONGATEWAYS) | AGWFirewallLogs | ❌ Not Configured |
| ApplicationGatewayPerformanceLog | AzureDiagnostics (APPLICATIONGATEWAYS) | AGWPerformanceLogs | ❌ Not Configured |

---

## Azure Firewall

**Resource Provider:** `Microsoft.Network/azureFirewalls`  
**Status:** ✅ Complete — validated 2026-03-02  
**Wiki Entry:** `Application rule logs, network rule logs, DNS proxy logs, IDPS signature logs` → **4 entries (1-for-1)**  
**Diagnostic Mode:** Azure Diagnostics (legacy + structured category names simultaneously)  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Azure Firewall structured logs](https://learn.microsoft.com/azure/firewall/firewall-structured-logs) | [Supported logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-network-azurefirewalls-logs)

### What We Did

1. Wiki lists 4 log types for Firewall — workbook tracks exactly those 4
2. Resource-specific tables (AZFWApplicationRule, etc.) all returned **0 records** — customer uses AzureDiagnostics mode, NOT resource-specific
3. AzureDiagnostics confirmed ingesting with **both legacy and structured category names** simultaneously (3 of 4):
   - Application Rule logs → AzureFirewallApplicationRule: 292M all-time
   - Network Rule logs → AzureFirewallNetworkRule: 34M all-time
   - DNS Proxy logs → AzureFirewallDnsProxy: 102M all-time
4. IDPS Signature logs → AZFWIdpsSignature: **0 records** — IDPS is an Azure Firewall **Premium** feature; not enabled on this firewall SKU/policy
5. Previously had 7 entries (included ThreatIntel, NatRule, FlowTrace) — **removed 3 extras** not specified in wiki to be 1-for-1
6. Source subscription: `ac95a806-c9d3-49e7-83ee-7f82e88c2bd3`

### Table Status (4 entries in workbook — wiki 1-for-1)

| Wiki Log Type | CheckId | Resource-Specific Table | AZDIAG_FALLBACK Category | All-Time Count | Status |
|--------------|---------|------------------------|--------------------------|---------------|--------|
| Application rule logs | AZFWApplicationRule | AZFWApplicationRule (0) | AzureFirewallApplicationRule | 292M | ✅ Ingesting (via fallback) |
| Network rule logs | AZFWNetworkRule | AZFWNetworkRule (0) | AzureFirewallNetworkRule | 34M | ✅ Ingesting (via fallback) |
| DNS proxy logs | AZFWDnsQuery | AZFWDnsQuery (0) | AzureFirewallDnsProxy | 102M | ✅ Ingesting (via fallback) |
| IDPS signature logs | AZFWIdpsSignature | AZFWIdpsSignature (0) | AZFWIdpsSignature | 0 | ❌ Not enabled (Premium feature) |

### Other Categories in AzureDiagnostics (NOT in wiki — not tracked in workbook)

| Category | All-Time Count | Notes |
|----------|---------------|-------|
| AZFWThreatIntel | 101 | Present in AzDiag but not in wiki — removed from workbook |
| AZFWNatRule | 77 | Present in AzDiag but not in wiki — removed from workbook |
| AZFWFlowTrace | 345 | Present in AzDiag but not in wiki — removed from workbook |
| AZFWApplicationRuleAggregation | 38,902 | Aggregation summary, not security-relevant |
| AZFWNetworkRuleAggregation | 8,086 | Aggregation summary, not security-relevant |

### Notes

- Customer has BOTH legacy and structured log category names enabled simultaneously in their diagnostic settings (AzureDiagnostics mode)
- If customer migrates to resource-specific mode, data will land in the dedicated tables and AZDIAG_FALLBACK will no longer be needed
- **AZFWIdpsSignature:** IDPS is an Azure Firewall Premium feature — requires Premium SKU and IDPS mode enabled in firewall policy. Per [MS Learn](https://learn.microsoft.com/azure/firewall/premium-features#idps), IDPS detects malicious traffic matching known signatures. Not enabled = not a logging gap but a feature gap — recommend enabling for ATO compliance
- ThreatIntel, NatRule, FlowTrace removed from workbook — not listed in wiki. They exist in AzDiag with historical data but are not MS-ISR required

---

## Azure Key Vault

**Resource Provider:** `Microsoft.KeyVault/Vaults`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Azure Diagnostics (legacy)  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Azure Key Vault logging](https://learn.microsoft.com/azure/key-vault/general/logging) | [Monitoring Key Vault](https://learn.microsoft.com/azure/key-vault/general/monitor-key-vault)

### What We Did

1. Checked MS docs — Key Vault supports both resource-specific tables (`AZKVAuditLogs`, `AZKVPolicyEvaluationDetailsLogs`) and legacy `AzureDiagnostics`
2. Ran KQL against customer workspace (c3elz7) — found customer is using **legacy AzureDiagnostics mode**
3. Resource-specific tables `AZKVAuditLogs` and `AZKVPolicyEvaluationDetailsLogs` both returned **0 records**
4. AzureDiagnostics confirmed ingesting:
   - `AuditEvent` category: **286,207 records** (30d), actively ingesting from 3 subscriptions
5. `AzurePolicyEvaluationDetails` category: **No data** in current 30d range
6. Kept single `AzDiag_VAULTS` entry that queries `AzureDiagnostics | where ResourceType == 'VAULTS'`
7. Verified detail query output (Sample Data panel) returns correct AuditEvent data
8. Removed 2 zero-data resource-specific entries from workbook (KV went from 3 → 1)

### Source Subscriptions (3)

| Subscription ID | 14d Count |
|----------------|-----------|
| `ac95a806-c9d3-49e7-83ee-7f82e88c2bd3` | 69,447 |
| `787e871a-84ba-43be-86bf-86bd1e408a4a` | 49,888 |
| `07526f72-6689-42be-945f-bb6ad0214b71` | 14,004 |

### Category → Table Mapping (Final — 1 entry in workbook)

| Diagnostic Category | Table | Mode | Status |
|---------------------|-------|------|--------|
| AuditEvent | `AzureDiagnostics` (`ResourceType == 'VAULTS'`) | Legacy | ✅ 286,207 records (30d) |

### Note for Team

If customer migrates Key Vault to **resource-specific mode** in the future, re-add `AZKVAuditLogs` and `AZKVPolicyEvaluationDetailsLogs` entries and remove the `AzDiag_VAULTS` entry.

---

## Azure Load Balancer

**Resource Provider:** `Microsoft.Network/loadBalancers`  
**Status:** ✅ Complete — validated 2026-03-02  
**Wiki Entry:** `Health probe events, load balancer alert events` → **1 entry (single MS-supported category)**  
**Diagnostic Mode:** Resource-specific  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Supported LB logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-network-loadbalancers-logs) | [LB Health Event Logs](https://learn.microsoft.com/azure/load-balancer/load-balancer-health-event-logs)

### What We Did

1. Per [MS Learn](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-network-loadbalancers-logs), Load Balancer has **only 1 diagnostic category**: `LoadBalancerHealthEvent` → `ALBHealthEvent`
2. Wiki says "Health probe events, load balancer alert events" — both are event types within the single `ALBHealthEvent` table (health events include DataPathAvailabilityCritical, NoHealthyBackends, SnatPortExhaustion, etc.)
3. `ALBHealthEvent` schema exists in workspace but 0 records — either no health events have been generated or diagnostic setting not configured
4. No AZDIAG_FALLBACK needed — LB only supports resource-specific mode
5. **No changes needed — 1 entry is correct (1-for-1 with MS-supported categories)**

### Table Status (1 entry in workbook)

| Wiki Log Type | CheckId | Table | All-Time Count | Status |
|--------------|---------|-------|---------------|--------|
| Health probe events + alert events | ALBHealthEvent | `ALBHealthEvent` | 0 | ⚠️ Schema only, 0 data |

### Notes

- LB health events are generated only when there are actual health issues (probe failures, SNAT exhaustion, platform throttling)
- 0 records may be a positive sign — no health issues have occurred
- Per MS docs, can take up to 90 minutes for logs to begin appearing after diagnostic setting is configured

---

## Log Analytics Workspace

**Resource Provider:** `Microsoft.OperationalInsights/workspaces`  
**Status:** ✅ Complete — validated 2026-03-02  
**Wiki Entry:** `Workspace audit logs for query activity, data access, and workspace management operations` → **5 entries (3 LA diagnostic categories + 2 Sentinel audit tables)**  
**Diagnostic Mode:** Resource-specific  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Monitor Log Analytics workspaces](https://learn.microsoft.com/azure/azure-monitor/logs/monitor-workspace) | [Supported logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-operationalinsights-workspaces-logs) | [Audit Sentinel queries](https://learn.microsoft.com/azure/sentinel/audit-sentinel-data)

### What We Did

1. Per [MS Learn](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-operationalinsights-workspaces-logs), LA Workspace has **3 diagnostic categories**: Audit, Jobs, SummaryLogs
2. Previously had only 1 entry (LAQueryLogs) — **added LAJobLogs and LASummaryLogs** to cover all MS-supported categories (best practice)
3. Wiki says "query activity, data access, and workspace management operations" — maps to all 3 categories:
   - **Query activity** → LAQueryLogs (Audit) — captures who ran what queries, when, from which app
   - **Data access** → LAJobLogs (Jobs) — tracks data export job executions
   - **Workspace management** → LASummaryLogs (SummaryLogs) — summary rule execution details
4. User confirmed auditing was just turned on — LAQueryLogs actively ingesting since 2026-02-01
5. Per [Sentinel audit docs](https://learn.microsoft.com/azure/sentinel/audit-sentinel-data), LAQueryLogs is critical for SOC audit trail — captures all queries run in Sentinel Logs blade
6. Added **SentinelHealth** and **SentinelAudit** tables — these are Sentinel-native audit/health tables (not LA diagnostic categories) but grouped under this workload since wiki bundles everything under `loganalytics`
7. Per [MS Learn](https://learn.microsoft.com/azure/sentinel/health-audit), SentinelHealth monitors data connector health, analytics rule execution, and automation (NOT billable). SentinelAudit tracks resource changes (billable).

### Table Status (5 entries in workbook)

| Wiki Mapping | CheckId | Table | All-Time Count | 30d Count | Last Record | Status |
|-------------|---------|-------|---------------|-----------|-------------|--------|
| Query activity | LAQueryLogs | `LAQueryLogs` | 19,707 | 1,535 | 2026-03-02 | ✅ Ingesting |
| Data access | LAJobLogs | `LAJobLogs` | 0 | 0 | — | ℹ️ Schema only |
| Workspace management | LASummaryLogs | `LASummaryLogs` | 0 | 0 | — | ℹ️ Schema only |
| Sentinel health monitoring | SentinelHealth | `SentinelHealth` | — | — | 2026-03-02 | ✅ Ingesting |
| Sentinel audit trail | SentinelAudit | `SentinelAudit` | — | — | 2026-03-02 | ✅ Ingesting |

### Notes

- **LAQueryLogs** is the most security-relevant — required for ATO audit trail. Captures who queried what data in Sentinel/Log Analytics.
- **LAJobLogs** tracks export jobs (data exfiltration detection) — will populate when export jobs run
- **LASummaryLogs** tracks summary rule execution — will populate when summary rules are configured
- **SentinelHealth** monitors data connector health, analytics rule health, and automation health — NOT billable per MS Learn
- **SentinelAudit** tracks Sentinel resource changes (analytics rules, data connectors, workbooks, etc.) — billable
- Auditing was enabled ~2026-02-01 per first record timestamp
- Per MS Learn, LAQueryLogs only captures queries from the Sentinel Logs blade, NOT from scheduled analytics rules, Investigation Graph, or Hunting page

---

## Network Interface

**Resource Provider:** `Microsoft.Network/networkInterfaces`  
**Status:** ✅ Complete — validated 2026-03-02  
**Wiki Entry:** `Effective network security group rules and flow data (supplementary to NSG flow logs)` → **1 entry (wiki marker — no diagnostic categories exist)**  
**Diagnostic Mode:** N/A — **No diagnostic log categories exist for NICs**  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [NIC monitoring data reference](https://learn.microsoft.com/azure/virtual-network/monitor-virtual-network-reference) | [LA tables for NICs](https://learn.microsoft.com/azure/azure-monitor/reference/tables/microsoft-network_networkinterfaces)

### What We Did

1. Searched MS Learn for `microsoft.network/networkInterfaces` — **confirmed NICs have NO diagnostic log categories** (only `AzureActivity` + `AzureMetrics`)
2. Verified AzureDiagnostics NETWORKINTERFACES = 0 records (expected)
3. Verified tenant has 161 NIC resources — none can send diagnostic logs (by design)
4. Wiki says "Effective network security group rules and flow data (supplementary to NSG flow logs)" — this is misleading. NIC-level flow visibility comes from:
   - **NSG Diagnostic Settings** → `AzureDiagnostics` (tracked under NSG workload)
   - **VNET Flow Logs** → `NTANetAnalytics` + `AzureNetworkAnalytics_CL` (tracked under VNET Flow Logs workload)
5. Workbook entry `AzDiag_NETWORKINTERFACES` kept as wiki completeness marker — always shows "Not Configured"

### How NIC Monitoring Actually Works

```
NIC itself → NO diagnostic logs (by design)
  ↕ Associated NSG
NSG Diagnostic Settings → AzureDiagnostics (rule events + counters) → tracked under "Network Security Group" workload
  ↕ VNet the NIC lives on
Network Watcher Flow Logs → Storage → Traffic Analytics → NTANetAnalytics / AzureNetworkAnalytics_CL → tracked under "VNET Flow Logs" workload
```

### Table Status (1 entry in workbook)

| Wiki Mapping | CheckId | Table | Status |
|-------------|---------|-------|--------|
| Effective NSG rules + flow data | AzDiag_NETWORKINTERFACES | `AzureDiagnostics` | ℹ️ Not Configured (expected — no log categories exist) |

### Notes

- This is NOT an ATO gap — it's by design. NIC flow visibility is covered by NSG + VNET Flow Logs workloads.
- The wiki description is misleading — NSG rules and flow data do not originate from NIC diagnostic settings.
- 161 NICs exist in tenant; none can send diagnostic logs per MS Learn.

---

## Network Security Group

**Resource Provider:** `Microsoft.Network/networkSecurityGroups`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Azure Diagnostics (legacy)  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)

### What We Did

1. Queried AzureDiagnostics for ResourceType NETWORKSECURITYGROUPS
2. Found **3.5M records in 30d** across 2 categories, 2 subscriptions — actively ingesting
3. All-time: 44M+ records across 5 subscriptions

### Live Data (30d)

| Category | Records (30d) | Last Record | Subs (30d) |
|----------|---------------|-------------|------------|
| NetworkSecurityGroupEvent | 1,742,670 | 2026-03-02 | 2 |
| NetworkSecurityGroupRuleCounter | 1,742,657 | 2026-03-02 | 2 |
| **Total** | **3,485,327** | | |

### Subscription Breakdown (30d)

| Subscription ID | Records |
|-----------------|---------|
| `ac95a806-c9d3-49e7-83ee-7f82e88c2bd3` | 2,607,777 |
| `787e871a-84ba-43be-86bf-86bd1e408a4a` | 877,349 |

### Category → Table Mapping (Final — 1 entry in workbook)

| Diagnostic Category | Table | NIST Controls | Status |
|---------------------|-------|---------------|--------|
| NetworkSecurityGroupEvent + NetworkSecurityGroupRuleCounter | `AzDiag_NETWORKSECURITYGROUPS` | AC-4, SC-7, AU-2, SI-4 | ✅ Ingesting |

### Notes

- NSG diagnostics use AzureDiagnostics (legacy mode) — no resource-specific tables available.
- Massive data volume (3.5M/30d). Healthy ingestion across 2 subscriptions.
- All-time data shows 5 subscriptions, but only 2 active in last 30d. The other 3 may have stopped sending.

---

## Public IP Address

**Resource Provider:** `Microsoft.Network/publicIPAddresses`  
**Status:** ✅ Complete — validated 2026-03-02  
**Wiki Entry:** `DDoS mitigation reports, DDoS protection alert logs (where applicable)` → **3 entries (all 3 MS Learn DDoS categories)**  
**Diagnostic Mode:** Azure Diagnostics (legacy)  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Supported PIP logs](https://learn.microsoft.com/azure/azure-monitor/reference/supported-logs/microsoft-network-publicipaddresses-logs) | [DDoS diagnostic logging](https://learn.microsoft.com/azure/ddos-protection/diagnostic-logging) | [View DDoS logs in LA](https://learn.microsoft.com/azure/ddos-protection/ddos-view-diagnostic-logs)

### How It's Turned On

- **Diagnostic settings** on each Public IP resource via Azure Policy (`setByPolicyEventhub`)
- Verified on `gatesdeployVNETip229`: All 3 DDoS categories enabled
- ⚠️ **ISSUE: Destination is Event Hub ONLY** — not sending to Log Analytics Workspace
  - `eventHubAuthorizationRuleId` → `c3elz7-eventhub-ns/RootManageSharedAccessKey`
  - `eventHubName` → `rglogs-eventhub`
  - `workspaceId` → *(empty)*
- ⚠️ **ISSUE: No DDoS Protection Plan exists in tenant** — even if logs were sent to LA, they'd be empty without a plan
- 4 Public IPs exist across 2 subs (gates20, jb-policy-testing)

### MS Learn Categories vs Wiki

| MS Learn Category | Wiki Mapping | Enabled | Destination | Data |
|-------------------|-------------|---------|-------------|------|
| `DDoSProtectionNotifications` | DDoS protection alert logs | ✅ Yes | Event Hub only | 0 (no DDoS Plan) |
| `DDoSMitigationFlowLogs` | DDoS mitigation reports | ✅ Yes | Event Hub only | 0 (no DDoS Plan) |
| `DDoSMitigationReports` | DDoS mitigation reports | ✅ Yes | Event Hub only | 0 (no DDoS Plan) |

### What Would Need to Happen for Data to Appear

1. **Deploy Azure DDoS Protection Plan** and associate it with VNets containing Public IPs
2. **Add Log Analytics Workspace destination** to the diagnostic settings (currently Event Hub only)
3. Data would only appear during actual DDoS attacks (mitigation events)
4. Wiki says "where applicable" — acknowledges DDoS may not be deployed

### Table Status (3 entries in workbook)

| Wiki Mapping | CheckId | AzDiag Category Filter | All-Time Count | Status |
|-------------|---------|----------------------|---------------|--------|
| DDoS protection alerts | AzDiag_PIP_DDoSNotify | `PUBLICIPADDRESSES:DDoSProtectionNotifications` | 0 | ⚠️ Event Hub only, not LA |
| DDoS mitigation flows | AzDiag_PIP_DDoSFlow | `PUBLICIPADDRESSES:DDoSMitigationFlowLogs` | 0 | ⚠️ Event Hub only, not LA |
| DDoS mitigation reports | AzDiag_PIP_DDoSReport | `PUBLICIPADDRESSES:DDoSMitigationReports` | 0 | ⚠️ Event Hub only, not LA |

### Notes

- All 3 categories are DDoS-related — they require Azure DDoS Protection Plan to generate data
- Diagnostic settings ARE configured via policy but only to Event Hub, NOT to Log Analytics
- For ATO: If DDoS Protection is in scope, two infrastructure changes are needed: (1) deploy DDoS Plan, (2) add LA workspace destination to diag settings
- The wiki's "where applicable" qualifier suggests this may be an accepted risk if DDoS Protection is out of scope

---

## Azure Recovery Services Vault

**Resource Provider:** `Microsoft.RecoveryServices/Vaults`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Resource-specific  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [Diagnostic Events for Azure Backup](https://learn.microsoft.com/azure/backup/backup-azure-diagnostic-events) | [Monitor Site Recovery](https://learn.microsoft.com/azure/site-recovery/monitor-log-analytics)

### What We Did

1. Cross-referenced MS docs for all RSV diagnostic categories and expected tables
2. Ran KQL against customer workspace (c3elz7) — confirmed 6 backup tables ingesting
3. Confirmed 3 entries have schema but no data: `AzureBackupOperations`, `ASRJobs`, `ASRReplicatedItems`
4. Kept all 9 entries in workbook — schema-only tables show "Configured - No Activity" (ATO Pass)
5. Fixed AzureDiagnostics false positive — `ResourceType == 'VAULTS'` was colliding with Key Vault; resolved with `_azdiag_filter()` category-level filtering
6. Added AZDIAG_FALLBACK entries for ASR tables
7. Verified detail query output (Sample Data panel) returns correct data for all ingesting tables

### Required Diagnostic Settings

**One diagnostic setting in Resource-specific mode** covering 6 backup categories + 2 ASR categories. ASR tables show schema but no replication activity (expected — no Site Recovery configured).

### Category → Table Mapping (Final — 9 entries in workbook)

| Diagnostic Category | Resource-Specific Table | NIST Controls | Status |
|---------------------|------------------------|---------------|--------|
| Core Azure Backup Data | `CoreAzureBackup` | CP-9, CP-10, AU-2 | ✅ Ingesting |
| Addon Azure Backup Job Data | `AddonAzureBackupJobs` | CP-9, CP-10, AU-2 | ✅ Ingesting |
| Addon Azure Backup Alert Data | `AddonAzureBackupAlerts` | CP-9, CP-10, SI-4 | ✅ Ingesting |
| Addon Azure Backup Policy Data | `AddonAzureBackupPolicy` | CP-9, CP-10, CM-6 | ✅ Ingesting |
| Addon Azure Backup Storage Data | `AddonAzureBackupStorage` | CP-9, CP-10, AU-2 | ✅ Ingesting |
| Addon Azure Backup Protected Instance Data | `AddonAzureBackupProtectedInstance` | CP-9, CP-10, CM-8 | ✅ Ingesting |
| Azure Backup Operations | `AzureBackupOperations` | CP-9, CP-10, AU-2, AU-12 | ℹ️ Schema only, no data |
| Site Recovery Jobs | `ASRJobs` | CP-9, CP-10, IR-4, AU-2 | ℹ️ Schema only, no data |
| Site Recovery Replicated Items | `ASRReplicatedItems` | CP-9, CP-10, CM-8, SI-4 | ℹ️ Schema only, no data |

### AZDIAG_FALLBACK entries (in generator)

```python
"CoreAzureBackup": ("VAULTS", "CoreAzureBackup"),
"AddonAzureBackupJobs": ("VAULTS", "AddonAzureBackupJobs"),
"AddonAzureBackupPolicy": ("VAULTS", "AddonAzureBackupPolicy"),
"AddonAzureBackupStorage": ("VAULTS", "AddonAzureBackupStorage"),
"AddonAzureBackupAlerts": ("VAULTS", "AddonAzureBackupAlerts"),
"AddonAzureBackupProtectedInstance": ("VAULTS", "AddonAzureBackupProtectedInstance"),
"ASRJobs": ("VAULTS", "AzureSiteRecoveryJobs"),
"ASRReplicatedItems": ("VAULTS", "AzureSiteRecoveryReplicatedItems"),
```

### Validation KQL

```kql
union isfuzzy=true
    (CoreAzureBackup | summarize Count=count(), Last=max(TimeGenerated) | extend Table="CoreAzureBackup"),
    (AddonAzureBackupJobs | summarize Count=count(), Last=max(TimeGenerated) | extend Table="AddonAzureBackupJobs"),
    (AddonAzureBackupAlerts | summarize Count=count(), Last=max(TimeGenerated) | extend Table="AddonAzureBackupAlerts"),
    (AddonAzureBackupPolicy | summarize Count=count(), Last=max(TimeGenerated) | extend Table="AddonAzureBackupPolicy"),
    (AddonAzureBackupStorage | summarize Count=count(), Last=max(TimeGenerated) | extend Table="AddonAzureBackupStorage"),
    (AddonAzureBackupProtectedInstance | summarize Count=count(), Last=max(TimeGenerated) | extend Table="AddonAzureBackupProtectedInstance")
| project Table, Count, Last
| sort by Table asc
```

---

## Azure Cognitive Search

**Resource Provider:** `Microsoft.Search/searchServices`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Azure Diagnostics (legacy)  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)

### What We Did

1. Queried AzureDiagnostics for ResourceType SEARCHSERVICES
2. Found **624 records in 30d** — actively ingesting from 1 subscription
3. Single category: `OperationLogs`

### Live Data (30d)

| Category | Records (30d) | Last Record | Subs |
|----------|---------------|-------------|------|
| OperationLogs | 624 | 2026-03-02 | 1 |

### Subscription Breakdown (30d)

| Subscription ID | Records |
|-----------------|---------|
| `ac95a806-c9d3-49e7-83ee-7f82e88c2bd3` | 624 |

### Category → Table Mapping (Final — 1 entry in workbook)

| Diagnostic Category | Table | NIST Controls | Status |
|---------------------|-------|---------------|--------|
| OperationLogs | `AzDiag_SEARCHSERVICES` | AU-2, AU-3, AU-12 | ✅ Ingesting |

---

## Virtual Network Gateway

**Resource Provider:** `Microsoft.Network/virtualNetworkGateways`  
**Status:** ✅ Complete — validated 2026-03-02  
**Diagnostic Mode:** Azure Diagnostics (legacy)  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [VPN Gateway monitoring data reference](https://learn.microsoft.com/azure/vpn-gateway/monitor-vpn-gateway-reference)

### What We Did

1. Wiki abbreviation "VNetGW" refers to traditional Virtual Network Gateways (`microsoft.network/virtualNetworkGateways`), **not** Virtual WAN's VPN Gateway (`microsoft.network/vpnGateways`)
2. Changed workbook ResourceFilter from `VPNGATEWAYS` to `VIRTUALNETWORKGATEWAYS` to match wiki intent
3. Queried AzureDiagnostics for both `VPNGATEWAYS` and `VIRTUALNETWORKGATEWAYS` — **0 records** for both (all-time)
4. Entry shows "Not Configured" — expected if no VPN/ExpressRoute gateways have diagnostic settings configured

### Category → Table Mapping (Final — 1 entry in workbook)

| Diagnostic Category | Table | NIST Controls | Status |
|---------------------|-------|---------------|--------|
| GatewayDiagnosticLog, TunnelDiagnosticLog, RouteDiagnosticLog, IKEDiagnosticLog, P2SDiagnosticLog | `AzDiag_VIRTUALNETWORKGATEWAYS` | AC-17, SC-7, AU-2 | ❌ Not Configured |

### Notes

- 0 records all-time for both VIRTUALNETWORKGATEWAYS and VPNGATEWAYS resource types.
- Customer may not have VPN/ExpressRoute gateways deployed, or diagnostic settings may not be configured.
- If VNet gateways exist, this is a **gap** — confirm with customer.

---

## VNET Flow Logs

**Status:** ✅ Complete — validated 2026-03-02  
**Wiki Entry:** `Captures source/destination IP, port, protocol, traffic direction, and allow/deny disposition for all flows traversing VNets` → **2 entries**  
**Diagnostic Mode:** Network Watcher → Storage → Traffic Analytics → Log Analytics  
**Workspace:** `log-c3elz7-usgovvirginia-001` (ID: `5605c9c4-0f2a-49f5-ae19-d7f8af033df7`)  
**MS Docs:** [VNET Flow Logs](https://learn.microsoft.com/azure/network-watcher/vnet-flow-logs-overview) | [Traffic Analytics](https://learn.microsoft.com/azure/network-watcher/traffic-analytics)

### How It's Turned On

- **Network Watcher flow log configurations** on VNets and/or NSGs
- NOT diagnostic settings — this is a separate mechanism via Network Watcher
- 20+ flow log configs verified in tenant via Resource Graph (`microsoft.network/networkwatchers/flowlogs`)
- Flow data pipeline: `VNet/NSG → Network Watcher → Storage Account → Traffic Analytics → Log Analytics`
- Both VNet-level and NSG-level flow configs exist (VNet-level is newer, NSG-level is legacy)

### Architecture: How VNET Flow Logs Relate to Other Networking Workloads

```
┌─────────────────────────────────────────────────────────────────────┐
│ MECHANISM 1: Diagnostic Settings (per-resource)                    │
│                                                                     │
│  NIC → No diagnostic categories (by design)                        │
│  NSG → AzureDiagnostics (rule events + counters) → Log Analytics   │
│  PIP → AzureDiagnostics (DDoS categories) → Event Hub only         │
├─────────────────────────────────────────────────────────────────────┤
│ MECHANISM 2: Network Watcher Flow Logs (this workload)             │
│                                                                     │
│  VNet/NSG → Network Watcher → Storage → Traffic Analytics          │
│          → NTANetAnalytics (new) + AzureNetworkAnalytics_CL (old)  │
│          → Captures actual 5-tuple flow data (src/dst/port/proto)  │
└─────────────────────────────────────────────────────────────────────┘
```

### Live Data (all-time)

| Table | All-Time Count | Last Record | Status |
|-------|---------------|-------------|--------|
| `NTANetAnalytics` | 2,951,455 | 2026-03-02 | ✅ Ingesting |
| `AzureNetworkAnalytics_CL` | 4,539,476 | 2026-03-02 | ✅ Ingesting |
| **Total** | **~7.5M** | | |

### Table Status (2 entries in workbook)

| Wiki Mapping | CheckId | Table | All-Time Count | Status |
|-------------|---------|-------|---------------|--------|
| VNet flow data (new pipeline) | NTANetAnalytics | `NTANetAnalytics` | 2.9M | ✅ Ingesting |
| VNet flow data (legacy pipeline) | AzureNetworkAnalytics_CL | `AzureNetworkAnalytics_CL` | 4.5M | ✅ Ingesting |

### Notes

- `NTANetAnalytics` is the newer resource-specific table via VNET Flow Logs
- `AzureNetworkAnalytics_CL` is the legacy custom log table via NSG Flow Logs + Traffic Analytics
- Both are actively ingesting — tenant has both old (NSG-level) and new (VNet-level) flow log configs
- 20+ Network Watcher flow log configurations active across multiple VNets and NSGs
- Combined volume of ~7.5M records indicates comprehensive flow telemetry
- This workload provides the actual flow-level visibility that the NIC and NSG wiki entries reference
