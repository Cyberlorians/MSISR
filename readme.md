# ATO Log Source Validation Tracker

> **Purpose:** Track every Azure resource workload from the MS-ISR Default Logging Configuration wiki — what was expected, what we deployed in the Sentinel workbook, and whether data is flowing.  
> **Wiki Source:** Azure DevOps — MSISR / Default Logging Configuration (John Blevins)  
> **Workbook:** `generate_workbook.py` → `msisr.json` → `deploy_workbook.ps1`  
> **Workspace:** `log-c3elz7-usgovvirginia-001`  
> **Last Updated:** 2026-03-03  
> **Total Entries:** 70 across 22 workloads

---

## Status Legend

| Status | Meaning | ATO Impact |
|--------|---------|------------|
| ✅ Ingesting | Data actively flowing to Log Analytics | Pass |
| ℹ️ Configured | Table exists in workspace schema, no data in time range | Pass (logging is on, no events occurred) |
| ⚠️ Needs Review | Historical data exists but nothing recent — customer follow-up required | Pending |
| ❌ Not Configured | Table does not exist in workspace | Finding |

---

## Summary

| # | Wiki Abbrev | Workload | Entries | Status | Source |
|---|-------------|----------|---------|--------|--------|
| 1 | — | Entra ID | 12 | ✅ Complete | Tenant |
| 2 | — | Azure Activity | 1 | ✅ Complete | 3 Subscriptions |
| 3 | AA | Azure Automation Account | 1 | ✅ Complete | Platform Mgmt Sub |
| 4 | ACR | Azure Container Registry | 2 | ✅ Complete | Platform Mgmt Sub |
| 5 | ADT | Azure Data Transfer | 1 | ⚠️ Needs Review | Platform Mgmt Sub |
| 6 | AKS | Azure Kubernetes Service | 1 | ⚠️ Needs Review | Multiple Subs |
| 7 | APIMgmt | Azure API Management | 1 | ⚠️ Needs Review | Platform Mgmt Sub |
| 8 | AppGroup | AVD AppGroup | 3 | ✅ Complete | 2 Subscriptions |
| 9 | ApplicationGateway | Azure Application Gateway | 3 | ✅ Complete | No resources deployed |
| 10 | Firewall | Azure Firewall | 4 | ✅ Complete | Platform Mgmt Sub |
| 11 | HostPool | Azure Virtual Desktop | 11 | ✅ Complete | 2 Subscriptions |
| 12 | KeyVault | Azure Key Vault | 1 | ✅ Complete | 3 Subscriptions |
| 13 | LoadBalancer | Azure Load Balancer | 1 | ✅ Complete | Schema only |
| 14 | loganalytics | Log Analytics Workspace | 5 | ✅ Complete | Tenant + SIEM Sub |
| 15 | NIC | Network Interface | 1 | ✅ Complete | AzureMetrics (19 NICs) |
| 16 | NSG | Network Security Group | 2 | ✅ Complete | 2 Subscriptions |
| 17 | PublicIP | Public IP Address | 3 | ✅ Complete | Event Hub only (not LA) |
| 18 | RecoveryVault | Azure Recovery Services Vault | 9 | ✅ Complete | Multiple Subs |
| 19 | SearchServices | Azure Cognitive Search | 1 | ✅ Complete | Platform Mgmt Sub |
| 20 | VNetGW | Virtual Network Gateway | 1 | ✅ Complete | ❌ Not Configured |
| 21 | Workspace | AVD Workspace | 4 | ✅ Complete | 2 Subscriptions |
| 22 | — | VNET Flow Logs | 2 | ✅ Complete | Multiple Subs |

---

## Customer Follow-Up Items

| Workload | Issue | Recommendation |
|----------|-------|----------------|
| Azure Data Transfer | Last record 2024-07-04, no recent data | Confirm resource is still active |
| Azure Kubernetes Service | Last record 2025-12-09, 0 in 30d | Confirm AKS clusters are still in use |
| Azure API Management | Only 2 records ever (2024-03-20) | Confirm APIM is deployed and diag settings configured |
| Public IP Address | Diag settings send to Event Hub only, not LA | Add LA workspace as destination; deploy DDoS Protection Plan if in scope |
| Virtual Network Gateway | 0 records all-time | Confirm VPN/ExpressRoute gateways exist; enable diagnostic settings |
| Azure Firewall (IDPS) | AZFWIdpsSignature = 0 | IDPS requires Firewall Premium SKU — enable if in scope |

---

## Workload Details

<details>
<summary><strong>1. Entra ID</strong> — 12 entries — ✅ Complete</summary>

### Wiki Said
Entra ID diagnostic logs including sign-ins, audit, provisioning, identity protection risk events, Graph activity, and ADFS.

### What We Deployed
12 dedicated table entries covering all Entra ID log categories. These are tenant-level logs — source shows "Tenant" (no subscription extraction).

| Table | Display Name | Controls | Status | Notes |
|-------|-------------|----------|--------|-------|
| SigninLogs | Sign-in Logs | AC-2, IA-2, AU-2, AU-3 | ✅ Ingesting | 81K records (30d) |
| AADNonInteractiveUserSignInLogs | Non-Interactive Sign-ins | AC-2, IA-2, AU-2, AU-3 | ℹ️ Configured | Schema only — interactive auth may be primary |
| AuditLogs | Audit Logs | AC-2, AC-6, AU-2, AU-3, AU-12 | ✅ Ingesting | 20K+ records (30d) |
| AADServicePrincipalSignInLogs | Service Principal Sign-ins | AC-2, IA-2, AU-2 | ✅ Ingesting | 1.9M records (30d) |
| AADManagedIdentitySignInLogs | Managed Identity Sign-ins | AC-2, IA-2, IA-4, AU-2 | ✅ Ingesting | 6M records (30d) |
| AADProvisioningLogs | Provisioning Logs | AC-2, AU-2, AU-12 | ℹ️ Configured | No provisioning connectors actively syncing |
| AADRiskyUsers | Risky Users | AC-2, IA-5, SI-4, RA-5 | ✅ Ingesting | Low volume (47 in 30d) |
| AADUserRiskEvents | User Risk Events | AC-2, IA-5, SI-4, RA-5 | ✅ Ingesting | Low volume (58 in 30d) |
| AADRiskyServicePrincipals | Risky Service Principals | AC-2, IA-5, SI-4 | ℹ️ Configured | No SP risk detections |
| AADServicePrincipalRiskEvents | SP Risk Events | AC-2, IA-5, SI-4, RA-5 | ℹ️ Configured | No SP risk detections |
| ADFSSignInLogs | ADFS Sign-in Logs | AC-2, IA-2, AU-2 | ℹ️ Configured | No ADFS federation configured |
| MicrosoftGraphActivityLogs | Graph Activity Logs | AC-2, AC-6, AU-2, AU-3, SI-4 | ✅ Ingesting | 10.6M records (30d) |

**Wiki Cross-Reference:** NonInteractiveUserSignInLogs and AuditLogs are not listed in the wiki but are actively ingesting — included as best practice.

</details>

<details>
<summary><strong>2. Azure Activity</strong> — 1 entry — ✅ Complete</summary>

### Wiki Said
Tenant Root Directory, Management Group, and Subscription Activity Logs.

### What We Deployed
Single `AzureActivity` table entry. Activity Logs from all three scopes (root, MG, subscription) flow into the same table.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureActivity | Activity Logs | AU-2, AU-3, AU-6, AU-12, CM-3 | ✅ Ingesting | 3 subscriptions, 87K+ records (30d) |

</details>

<details>
<summary><strong>3. Azure Automation Account</strong> (AA) — 1 entry — ✅ Complete</summary>

### Wiki Said
Automation Account diagnostic logs.

### What We Deployed
Single AzureDiagnostics entry filtering on `ResourceType == 'AUTOMATIONACCOUNTS'`. Covers all categories in one entry.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureDiagnostics (AUTOMATIONACCOUNTS) | Automation Diagnostics | CM-3, CM-6, AU-2, AU-12 | ✅ Ingesting | Platform Mgmt Sub |

**Categories flowing:** AuditEvent (990), JobLogs (2,973), JobStreams (42,815) in 30d. DscNodeStatus not configured (no DSC in use).

</details>

<details>
<summary><strong>4. Azure Container Registry</strong> (ACR) — 2 entries — ✅ Complete</summary>

### Wiki Said
Container Registry diagnostic logs.

### What We Deployed
2 resource-specific table entries. Customer uses resource-specific mode (not AzureDiagnostics).

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| ContainerRegistryLoginEvents | Login Events | AU-2, AU-3, CM-8, SI-4 | ✅ Ingesting | Platform Mgmt Sub, 2,092 records (30d) |
| ContainerRegistryRepositoryEvents | Repository Events | AU-2, AU-3, CM-8, SI-4 | ℹ️ Configured | Schema only — no push/pull events in range |

</details>

<details>
<summary><strong>5. Azure Data Transfer</strong> (ADT) — 1 entry — ⚠️ Needs Review</summary>

### Wiki Said
Azure Data Transfer operational logs.

### What We Deployed
Single resource-specific table entry.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| DataTransferOperations | Data Transfer Operations | AU-2, AU-3, SI-4, CM-3 | ⚠️ Needs Review | Platform Mgmt Sub |

**Issue:** 45,180 historical records exist (2024-03-20 to 2024-07-04) but no data in last 30 days. Resource may be inactive. Customer follow-up needed.

</details>

<details>
<summary><strong>6. Azure Kubernetes Service</strong> (AKS) — 1 entry — ⚠️ Needs Review</summary>

### Wiki Said
AKS diagnostic logs including kube-audit, kube-audit-admin, kube-apiserver, kube-controller-manager, kube-scheduler, cluster-autoscaler, guard.

### What We Deployed
Single AzureDiagnostics entry filtering on `ResourceType == 'MANAGEDCLUSTERS'`. Covers all categories in one entry.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureDiagnostics (MANAGEDCLUSTERS) | AKS Diagnostics | AU-2, AU-3, CM-6, SI-4 | ⚠️ Needs Review | Multiple Subs |

**Issue:** 122M historical records across 21 clusters, 11 categories. Last record 2025-12-09. Zero data in last 30 days. Customer follow-up needed — AKS clusters may have been decommissioned.

</details>

<details>
<summary><strong>7. Azure API Management</strong> (APIMgmt) — 1 entry — ⚠️ Needs Review</summary>

### Wiki Said
API Management gateway logs, developer portal audit logs, WebSocket connection logs.

### What We Deployed
Single AzureDiagnostics entry filtering on `ResourceType == 'SERVICE'`.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureDiagnostics (SERVICE) | APIM Diagnostics | AU-2, AU-3, AC-4, SI-4 | ⚠️ Needs Review | Unknown |

**Issue:** Only 2 records ever (GatewayLogs category, 2024-03-20). Customer follow-up needed — confirm APIM is still deployed and diagnostic settings are configured.

</details>

<details>
<summary><strong>8. AVD AppGroup</strong> (AppGroup) — 3 entries — ✅ Complete</summary>

### Wiki Said
AVD Application Group diagnostic logs (checkpoints, errors, management).

### What We Deployed
3 resource-specific WVD table entries filtered by `_ResourceId has 'applicationgroups'`. Each has AZDIAG_FALLBACK to `APPLICATIONGROUPS` resource type.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| WVDManagement | Management | CM-3, CM-6, AU-2 | ✅ Ingesting | 2 Subs, 89 records (30d) |
| WVDCheckpoints | Checkpoints | AU-2, AU-3, SI-4 | ℹ️ Configured | Schema only — AppGroup checkpoint events are rare |
| WVDErrors | Errors | AU-2, SI-4, SI-11 | ℹ️ Configured | Schema only — no AppGroup errors in range |

</details>

<details>
<summary><strong>9. Azure Application Gateway</strong> (ApplicationGateway) — 3 entries — ✅ Complete</summary>

### Wiki Said
Application Gateway access logs, firewall logs, performance logs.

### What We Deployed
3 resource-specific table entries with AZDIAG_FALLBACK to `APPLICATIONGATEWAYS`.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AGWAccessLogs | Access Logs | SC-7, AU-2, AU-3, SI-4 | ℹ️ Configured | Schema only |
| AGWPerformanceLogs | Performance Logs | SC-7, AU-2, SI-4 | ℹ️ Configured | Schema only |
| AGWFirewallLogs | Firewall Logs | SC-7, AU-2, SI-4, AC-4 | ℹ️ Configured | Schema only |

**Note:** No Application Gateway resources appear to be deployed. All 3 tables have schema but 0 records. Not an ATO gap if no AppGW exists.

</details>

<details>
<summary><strong>10. Azure Firewall</strong> (Firewall) — 4 entries — ✅ Complete</summary>

### Wiki Said
Application rule logs, network rule logs, DNS proxy logs, IDPS signature logs.

### What We Deployed
4 resource-specific table entries (1-for-1 with wiki). Each has AZDIAG_FALLBACK because customer uses AzureDiagnostics mode (legacy category names).

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AZFWApplicationRule | Application Rules | SC-7, AU-2, SI-4, AC-4 | ✅ Ingesting | Platform Mgmt Sub, 292M all-time (via AzDiag fallback) |
| AZFWNetworkRule | Network Rules | SC-7, AU-2, SI-4, AC-4 | ✅ Ingesting | Platform Mgmt Sub, 34M all-time (via AzDiag fallback) |
| AZFWDnsQuery | DNS Queries | SC-7, SC-20, SI-4 | ✅ Ingesting | Platform Mgmt Sub, 102M all-time (via AzDiag fallback) |
| AZFWIdpsSignature | IDPS Signatures | SC-7, SI-3, SI-4 | ❌ Not Configured | IDPS requires Firewall Premium SKU — not enabled |

**Note:** Customer uses AzureDiagnostics mode with both legacy and structured category names simultaneously. Resource-specific tables have 0 records — data flows via AZDIAG_FALLBACK only.

</details>

<details>
<summary><strong>11. Azure Virtual Desktop</strong> (HostPool) — 11 entries — ✅ Complete</summary>

### Wiki Said
Session host health, connection activity, agent logs — mapped to all 11 MS-supported HostPool diagnostic categories.

### What We Deployed
11 resource-specific WVD table entries. Customer uses resource-specific mode (0 AzureDiagnostics records for HOSTPOOLS).

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| WVDConnections | Connections | AC-2, AC-17, AU-2, AU-3 | ✅ Ingesting | 2 Subs |
| WVDCheckpoints | Checkpoints | AU-2, AU-3, SI-4 | ✅ Ingesting | 2 Subs |
| WVDErrors | Errors | AU-2, SI-4, SI-11 | ✅ Ingesting | 2 Subs |
| WVDManagement | Management | CM-3, CM-6, AU-2 | ✅ Ingesting | 2 Subs |
| WVDHostRegistrations | Host Registrations | CM-3, CM-8, AU-2 | ✅ Ingesting | Low volume |
| WVDAgentHealthStatus | Agent Health Status | SI-4, CM-8, AU-2 | ✅ Ingesting | 1.2M all-time |
| WVDConnectionNetworkData | Connection Network Data | AC-17, SC-7, AU-2 | ℹ️ Configured | Historical only (last 2025-11-12) |
| WVDConnectionGraphicsDataPreview | Connection Graphics Preview | AC-17, AU-2 | ℹ️ Configured | Schema only |
| WVDSessionHostManagement | Session Host Mgmt | CM-3, CM-6, AU-2 | ℹ️ Configured | Schema only |
| WVDAutoscaleEvaluationPooled | Autoscale Pooled | CM-6, AU-2 | ℹ️ Configured | Schema only — no autoscale configured |
| WVDMultiLinkAdd | Multi-Link Add | AC-17, AU-2, SI-4 | ℹ️ Configured | Schema only |

**Note:** WVDFeeds removed from HostPool — per MS Learn, Feed is a Workspace-only diagnostic category. Tracked under AVD Workspace instead.

</details>

<details>
<summary><strong>12. Azure Key Vault</strong> (KeyVault) — 1 entry — ✅ Complete</summary>

### Wiki Said
Key Vault diagnostic logs (audit events).

### What We Deployed
Single AzureDiagnostics entry filtering on `ResourceType == 'VAULTS'`. Customer uses legacy mode — resource-specific tables (`AZKVAuditLogs`) have 0 records.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureDiagnostics (VAULTS) | Key Vault Diagnostics | AC-6, AU-2, AU-3, CM-6 | ✅ Ingesting | 3 Subs, 286K records (30d) |

</details>

<details>
<summary><strong>13. Azure Load Balancer</strong> (LoadBalancer) — 1 entry — ✅ Complete</summary>

### Wiki Said
Health probe events, load balancer alert events.

### What We Deployed
Single resource-specific table entry. MS Learn confirms Load Balancer has only 1 diagnostic category: `LoadBalancerHealthEvent` → `ALBHealthEvent`.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| ALBHealthEvent | Load Balancer Health | SC-7, AU-2, SI-4 | ℹ️ Configured | Schema only — 0 health events generated |

**Note:** 0 records may be positive — no health issues have occurred. LB health events fire only during probe failures, SNAT exhaustion, or platform throttling.

</details>

<details>
<summary><strong>14. Log Analytics Workspace</strong> (loganalytics) — 5 entries — ✅ Complete</summary>

### Wiki Said
Workspace audit logs for query activity, data access, and workspace management operations.

### What We Deployed
3 resource-specific LA diagnostic tables + 2 Sentinel audit tables. Wiki groups everything under `loganalytics` so we included Sentinel-native tables here.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| LAQueryLogs | Query Audit Logs | AU-2, AU-3, AU-6, AU-12 | ✅ Ingesting | SIEM Sub, 1,535 records (30d) |
| LAJobLogs | Job Logs | AU-2, AU-12 | ℹ️ Configured | Schema only — no export jobs running |
| LASummaryLogs | Summary Logs | AU-2, CM-6 | ℹ️ Configured | Schema only — no summary rules configured |
| SentinelHealth | Sentinel Health | SI-4, AU-2 | ✅ Ingesting | Connector/rule health monitoring (not billable) |
| SentinelAudit | Sentinel Audit | AU-2, AU-12, CM-3 | ✅ Ingesting | Resource change tracking (billable) |

**Note:** LAQueryLogs is critical for ATO audit trail — captures every query run in the Sentinel Logs blade. Auditing enabled ~2026-02-01.

</details>

<details>
<summary><strong>15. Network Interface</strong> (NIC) — 1 entry — ✅ Complete</summary>

### Wiki Said
Effective network security group rules and flow data (supplementary to NSG flow logs).

### What We Deployed
Single AzureMetrics entry. NICs have **no diagnostic log categories** — only AllMetrics is available. NIC flow visibility comes from NSG diagnostic settings and VNET Flow Logs (separate workloads).

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureMetrics (networkinterfaces) | NIC Metrics | AU-2, SI-4 | ✅ Ingesting | 19 NICs, 770K records (30d) |

**Metrics flowing:** BytesSentRate, BytesReceivedRate, PacketsSentRate, PacketsReceivedRate.

**Important:** This is not an ATO gap. The wiki description is misleading — NSG rules and flow data do not originate from NIC diagnostic settings. That data is covered by the NSG and VNET Flow Logs workloads.

</details>

<details>
<summary><strong>16. Network Security Group</strong> (NSG) — 2 entries — ✅ Complete</summary>

### Wiki Said
NSG diagnostic logs (event and rule counter categories).

### What We Deployed
2 AzureDiagnostics entries with category-level filtering. Deployed via Azure Policy (`setbypolicy`) to both LA and Event Hub.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureDiagnostics (NSG Event) | NSG Events | AC-4, SC-7, AU-2, SI-4 | ✅ Ingesting | 2 Subs, 1.7M records (30d) |
| AzureDiagnostics (NSG RuleCounter) | NSG Rule Counters | AC-4, SC-7, AU-2, SI-4 | ✅ Ingesting | 2 Subs, 1.7M records (30d) |

**Total:** 3.5M records (30d), 44M+ all-time across up to 5 subscriptions.

</details>

<details>
<summary><strong>17. Public IP Address</strong> (PublicIP) — 3 entries — ✅ Complete</summary>

### Wiki Said
DDoS mitigation reports, DDoS protection alert logs (where applicable).

### What We Deployed
3 AzureDiagnostics entries for the 3 DDoS diagnostic categories. Diag settings configured via Azure Policy.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureDiagnostics (PIP DDoSNotify) | DDoS Notifications | SC-5, SI-4, IR-4 | ❌ Not Configured | Event Hub only — not sent to LA |
| AzureDiagnostics (PIP DDoSFlow) | DDoS Mitigation Flows | SC-5, SI-4, AU-2 | ❌ Not Configured | Event Hub only — not sent to LA |
| AzureDiagnostics (PIP DDoSReport) | DDoS Mitigation Reports | SC-5, SI-4, AU-2 | ❌ Not Configured | Event Hub only — not sent to LA |

**Issues (2):**
1. Diagnostic settings destination is **Event Hub only** — LA workspace not configured as a destination
2. No **Azure DDoS Protection Plan** deployed — even with LA destination, no data would flow without a plan

**Wiki qualifier:** "where applicable" — may be an accepted risk if DDoS Protection is out of scope.

</details>

<details>
<summary><strong>18. Azure Recovery Services Vault</strong> (RecoveryVault) — 9 entries — ✅ Complete</summary>

### Wiki Said
Recovery Services Vault backup and site recovery diagnostic logs.

### What We Deployed
9 resource-specific table entries covering all backup and ASR diagnostic categories.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| CoreAzureBackup | Core Backup Data | CP-9, CP-10, AU-2 | ✅ Ingesting | Multiple Subs |
| AddonAzureBackupJobs | Backup Jobs | CP-9, CP-10, AU-2 | ✅ Ingesting | Multiple Subs |
| AddonAzureBackupAlerts | Backup Alerts | CP-9, CP-10, SI-4 | ✅ Ingesting | Multiple Subs |
| AddonAzureBackupPolicy | Backup Policy | CP-9, CP-10, CM-6 | ✅ Ingesting | Multiple Subs |
| AddonAzureBackupStorage | Backup Storage | CP-9, CP-10, AU-2 | ✅ Ingesting | Multiple Subs |
| AddonAzureBackupProtectedInstance | Protected Instances | CP-9, CP-10, CM-8 | ✅ Ingesting | Multiple Subs |
| AzureBackupOperations | Backup Operations | CP-9, CP-10, AU-2, AU-12 | ℹ️ Configured | Schema only |
| ASRJobs | Site Recovery Jobs | CP-9, CP-10, IR-4, AU-2 | ℹ️ Configured | Schema only — no ASR configured |
| ASRReplicatedItems | ASR Replicated Items | CP-9, CP-10, CM-8, SI-4 | ℹ️ Configured | Schema only — no ASR configured |

</details>

<details>
<summary><strong>19. Azure Cognitive Search</strong> (SearchServices) — 1 entry — ✅ Complete</summary>

### Wiki Said
Search Services diagnostic logs.

### What We Deployed
Single AzureDiagnostics entry filtering on `ResourceType == 'SEARCHSERVICES'`.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureDiagnostics (SEARCHSERVICES) | Search Diagnostics | AU-2, AU-3, AU-12 | ✅ Ingesting | Platform Mgmt Sub, 624 records (30d) |

</details>

<details>
<summary><strong>20. Virtual Network Gateway</strong> (VNetGW) — 1 entry — ✅ Complete</summary>

### Wiki Said
VPN Gateway diagnostic logs (gateway, tunnel, route, IKE, P2S diagnostics).

### What We Deployed
Single AzureDiagnostics entry filtering on `ResourceType == 'VIRTUALNETWORKGATEWAYS'`. Wiki abbreviation "VNetGW" refers to traditional Virtual Network Gateways, not Virtual WAN's VPN Gateway.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| AzureDiagnostics (VIRTUALNETWORKGATEWAYS) | VNet Gateway Diagnostics | AC-17, SC-7, AU-2 | ❌ Not Configured | 0 records all-time |

**Note:** ResourceFilter was corrected from `VPNGATEWAYS` to `VIRTUALNETWORKGATEWAYS`. Customer may not have VPN/ExpressRoute gateways deployed, or diagnostic settings are not configured.

</details>

<details>
<summary><strong>21. AVD Workspace</strong> (Workspace) — 4 entries — ✅ Complete</summary>

### Wiki Said
AVD Workspace diagnostic logs (feeds, management, checkpoints, errors).

### What We Deployed
4 resource-specific WVD table entries filtered by `_ResourceId has 'workspaces'`. Each has AZDIAG_FALLBACK to `WORKSPACES` resource type. All 4 are actively ingesting.

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| WVDFeeds | Feeds | AC-2, AU-2, CM-6 | ✅ Ingesting | 2 Subs, 1,634 records (30d) |
| WVDManagement | Management | CM-3, CM-6, AU-2 | ✅ Ingesting | 2 Subs, 1,006 records (30d) |
| WVDCheckpoints | Checkpoints | AU-2, AU-3, SI-4 | ✅ Ingesting | 2 Subs, 577 records (30d) |
| WVDErrors | Errors | AU-2, SI-4, SI-11 | ✅ Ingesting | 2 Subs, 8 records (30d) |

**Note:** WVDFeeds is a Workspace-only diagnostic category per MS Learn — not available on HostPool resources.

</details>

<details>
<summary><strong>22. VNET Flow Logs</strong> — 2 entries — ✅ Complete</summary>

### Wiki Said
Captures source/destination IP, port, protocol, traffic direction, and allow/deny disposition for all flows traversing VNets.

### What We Deployed
2 table entries covering both the new and legacy flow log pipelines. Data flows via Network Watcher → Storage → Traffic Analytics → Log Analytics (not via diagnostic settings).

| Table | Display Name | Controls | Status | Source |
|-------|-------------|----------|--------|--------|
| NTANetAnalytics | VNet Flow Analytics | AC-4, SC-7, AU-2, SI-4 | ✅ Ingesting | 2.9M all-time, 20+ flow configs |
| AzureNetworkAnalytics_CL | Legacy Flow Analytics | AC-4, SC-7, AU-2, SI-4 | ✅ Ingesting | 4.5M all-time, legacy NSG-level |

**Note:** `NTANetAnalytics` is the newer VNet-level flow log table. `AzureNetworkAnalytics_CL` is legacy (NSG-level). Both pipelines are active. Combined ~7.5M records all-time. This workload provides the actual flow-level visibility referenced by the NIC and NSG wiki entries.

</details>
