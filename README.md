# infra-ai-workshop

A Claude Code project providing two AI-assisted infrastructure reporting tools: AWS cost reporting and Terraform security review.

## Getting started

1. **Assume your AWS role** before opening Claude Code — the tools require active AWS credentials with the correct permissions.
2. Open this directory in Claude Code.
3. Tell Claude to **run the project**. It will confirm your role is assumed, then ask which tool to run.

---

## Tools

### 1. Cost Report

Queries AWS Cost Explorer and generates a polished HTML report showing production cloud spend for the last two full calendar months, broken down by region and team.

**At runtime Claude will ask:**
- Which region to report on — Production US (`us-east-1`), Production EU (`eu-west-1`), or both

**Tag filtering:**
- Team ownership identified by `Product` or `SubProduct` tag (values: `platform`, `humara`, `axiom`)
- Scoped to production only via the `Environment` tag (`live`, `prod`, or `production`)
- Resources with neither tag appear as `Untagged`

**What the report contains:**
- **Executive summary** — total spend, MoM delta, team snapshot, and a plain-English TL;DR paragraph calling out anomalies
- Regional split (US vs EU) when both regions are selected
- Per-team cards with month-over-month delta indicators
- Donut chart per region showing proportional team spend
- Top-5 services per team as horizontal bar charts
- Full service breakdown tables per team

**Requirements:** AWS credentials with `ce:GetCostAndUsage` and `sts:GetCallerIdentity`. The CE API is always queried via the `us-east-1` endpoint regardless of selected region.

---

### 2. Terraform Security Review

Statically analyses a Terraform codebase for HIGH and CRITICAL severity security issues and produces an HTML findings report. No AWS credentials required — analysis is purely local file reading.

**At runtime Claude will ask:**
- Which path to scan (default: `~/development`) — a tree of all Terraform projects found is shown for selection

**What it scans:**
- The selected project plus all referenced local modules and any cached remote modules (`.terraform/modules/`)
- 9 security categories: IAM, secrets & credentials, network exposure, encryption at rest, encryption in transit, logging & auditing, containers & serverless, resilience & safety, WAF

**What the report contains:**
- **Sticky nav bar** — CRITICAL and HIGH counts always visible while scrolling
- **Executive summary (TL;DR)** — overall risk level, stat bar, plain-English briefing for a non-technical reader, and a top-3 hit-list of the most critical findings
- **Findings summary table** — all findings in one sortable table with clickable links to detail cards
- **Per-service sections** — findings grouped by AWS service (S3, RDS, IAM, Lambda, etc.), ordered worst-first; each service has a severity bar showing the count of CRITICAL and HIGH findings at a glance
- **Per-finding detail cards** — resource, file, line, description, remediation callout, and a direct link to the relevant [Aqua AVD](https://avd.aquasec.com/misconfig/aws/) reference

---

## MCP server

Both tools use the **`aws-mcp`** MCP server defined in `.mcp.json`. All AWS API calls are routed through this server automatically. The server is pre-configured in `.claude/settings.local.json` with read-only tool permissions.

---

## Demo Terraform repo

A deliberately insecure Terraform project is available at `~/development/bad-terraform` for demonstrating the security review tool. It models a fictional payments platform ("Acme Payments") with intentional misconfigurations across all 9 security categories.

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
├── CLAUDE.md                         # Auto-loaded by Claude Code — persona, routing, MCP, rules
├── README.md                         # This file
├── cost-report-prompt.md             # Full instructions for the cost report tool
├── terraform-security-prompt.md      # Full instructions for the security review tool
├── .mcp.json                         # aws-mcp server configuration
├── .claude/
│   └── settings.local.json           # MCP server enablement + read-only tool allowlist
└── reports/
    ├── cost-report/                  # Generated cost reports
    └── terraform-security/           # Generated security reports
```

---

## Constraints

Both tools are **read-only**. No AWS resources, Terraform files, or any other content is created, modified, or deleted. The only writes are the HTML report files saved to the `reports/` directory.
