# infra-ai-workshop

A Claude Code project that provides two AI-assisted infrastructure reporting tools: AWS cost reporting and Terraform security review.

## Getting started

Open this directory in Claude Code. When you're ready, tell Claude to **run the project** and it will ask which tool you want to use.

---

## Tools

### 1. Cost Report

Queries AWS Cost Explorer and generates a styled HTML report showing cloud spend broken down by team tag (`platform`, `humara`, `axiom`).

**What it produces:**
- Total spend summary across all teams
- Per-team cost cards with top services breakdown
- Donut chart showing proportional spend
- Horizontal bar charts per team
- Saved to `reports/cost-report/`

**Requirements:** AWS credentials configured locally with Cost Explorer read access.

---

### 2. Terraform Security Review

Scans a Terraform codebase for HIGH and CRITICAL severity security issues and produces an HTML findings report.

**What it does:**
- Lists all Terraform projects found under `~/development` so you can pick one
- Scans the selected project plus any referenced local modules and cached remote modules
- Checks for issues across IAM, secrets, network exposure, encryption, logging, and more
- Reports only HIGH and CRITICAL findings, sorted by severity

**What it produces:**
- Executive summary with overall risk level
- Per-finding cards with file, line, description, and remediation advice
- Saved to `reports/terraform-security/`

**Requirements:** Read access to the target Terraform files. No AWS credentials required — analysis is purely static.

---

## Reports

```
reports/
├── cost-report/               # cost-report-YYYY-MM-HHmm.html
└── terraform-security/        # <project>-security-YYYY-MM-DD-HHmm.html
```

Reports are timestamped so multiple runs in the same day do not overwrite each other.

---

## Constraints

Both tools operate **read-only**. No AWS resources, Terraform files, or any other content is created, modified, or deleted. The only writes are the HTML report files saved to the `reports/` directory.
