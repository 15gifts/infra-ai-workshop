# infra-ai-workshop

A Claude Code project providing two AI-assisted infrastructure reporting tools: AWS cost reporting and Terraform security review.

## Getting started

Open this directory in Claude Code. Tell Claude to **run the project** and it will ask which tool you want to use.

---

## Tools

### 1. Cost Report

Queries AWS Cost Explorer and generates a styled HTML report showing cloud spend for the last two full calendar months, broken down by team tag.

**What it produces:**
- Grand total spend with month-over-month delta
- Per-team cards (`platform`, `humara`, `axiom`) with MoM delta indicators
- Donut chart showing proportional spend across teams
- Top-5 services per team as horizontal bar charts
- Full service breakdown tables
- Saved to `reports/cost-report/`

**Requirements:** AWS credentials configured locally with Cost Explorer read access (`ce:GetCostAndUsage`, `sts:GetCallerIdentity`).

---

### 2. Terraform Security Review

Scans a Terraform codebase statically for HIGH and CRITICAL security issues and produces an HTML findings report.

**What it does:**
- Lists all Terraform projects found under `~/development` — pick one by number
- Scans the selected project plus all referenced local modules and any cached remote modules (`.terraform/modules/`)
- Checks across 9 security categories: IAM, secrets, network exposure, encryption at rest, encryption in transit, logging, containers/serverless, resilience, and WAF
- Reports only HIGH and CRITICAL findings, with a summary table and per-finding detail cards

**What it produces:**
- Sticky nav bar with live finding counts
- Executive summary with overall risk level
- Clickable findings summary table (links to detail cards)
- Per-finding cards with file, line, resource, description, and remediation callout
- Saved to `reports/terraform-security/`

**Requirements:** Read access to the target Terraform files. No AWS credentials needed — analysis is purely static.

---

## Reports

```
reports/
├── cost-report/
│   └── cost-report-YYYY-MM-HHmm.html
└── terraform-security/
    └── <project-slug>-security-YYYY-MM-DD-HHmm.html
```

Reports are timestamped so multiple runs on the same day do not overwrite each other.

---

## Project structure

```
infra-ai-workshop/
├── CLAUDE.md                       # Loaded automatically by Claude Code — routes user intent
├── README.md                       # This file
├── cost-report-prompt.md           # Full instructions for the cost report tool
├── terraform-security-prompt.md    # Full instructions for the security review tool
└── reports/
    ├── cost-report/                # Generated cost reports
    └── terraform-security/         # Generated security reports
```

---

## Constraints

Both tools are **read-only**. No AWS resources, Terraform files, or any other content is created, modified, or deleted. The only writes are the HTML report files saved to the `reports/` directory.
