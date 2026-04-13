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

## Features

- **Coverage bar chart** — Stacked bar chart showing Ingesting (green), Configured (yellow), and Not Configured (red) counts per workload
- **Filterable parameters** — Subscription, Workspace, Time Range, Workload, NIST Control Family, Table, Status, and Audit Result
- **Hierarchical table** — Rows grouped by workload, expandable to individual log sources with status icons, last record timestamp, NIST control mappings, and direct links to Microsoft Learn documentation
- **Sample data panel** — Click any row to load the 50 most recent records from that table, with automatic actor/identity resolution (UPN, service principal, managed identity, or GUID → display name lookup)
- **Excel export** — Export the full table for offline evidence documentation

## Azure Workloads Covered

Entra ID, Azure Activity, Automation Account, Container Registry, Data Transfer, AKS, API Management, AVD (Host Pool, AppGroup, Workspace), Application Gateway, Azure Firewall, Key Vault, Load Balancer, Log Analytics / Sentinel, Network Interface, NSG, Public IP (DDoS), Recovery Services Vault (Backup + Site Recovery), Cognitive Search, Virtual Network Gateway, and VNET Flow Logs.

## NIST 800-53 Control Families

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

> ⚠️ **Disclaimer:** This workbook is an operational aid for validating log ingestion status. It does not replace your security assessment, System Security Plan (SSP), or formal ATO package requirements.
