# Terraform Security Review Prompt

## CRITICAL CONSTRAINT — READ-ONLY

**No write, create, update, or delete operations are permitted under any circumstances.**
All file access must be strictly read-only. Do not modify any Terraform files, state files, AWS resources, or any other content. The only permitted write is the HTML report output file.

---

## Task

Perform a static security review of a Terraform codebase located within `~/development`. Identify all **HIGH** and **CRITICAL** severity findings and produce a self-contained HTML report saved under `~/development/infra-ai-workshop/reports/terraform-security/`.

---

## Steps

### 1. Discover and present available Terraform projects

Run the following command to find all directories under `~/development` that contain at least one `.tf` file, excluding `.terraform/` cache directories:

```
find ~/development \( -name "*.tf" \) -not -path "*/.terraform/*" -type f \
  | sed 's|/[^/]*\.tf$||' | sort -u
```

If no results are returned, inform the user that no Terraform projects were found under `~/development` and stop.

Render the resulting directories as an indented tree grouped by path hierarchy. Number each entry. Example:

```
~/development/
├── 1. my-infra/terraform/prod
├── 2. my-infra/terraform/staging
├── 3. my-infra/modules/networking
└── 4. another-repo/infra
```

Present the tree and ask:

> Here are the Terraform projects found under `~/development`. Which one would you like to review? Enter the number or full path.

Wait for the user to choose before proceeding. Resolve the selection to its full absolute path and confirm the directory exists.

### 2. Discover all files to scan (read-only)

#### 2a. Find root project files

```
find <selected-path> \( -name "*.tf" -o -name "*.tfvars" -o -name "*.tfvars.json" \) \
  -not -path "*/.terraform/*" -type f | sort
```

#### 2b. Resolve module references

Read every `.tf` file and extract all `module` blocks. For each `source` value:

- **Local path** (`./` or `../` prefix): resolve relative to the declaring file. Recursively find all `.tf` files in that directory and add them to the scan scope. Follow any further module references found inside those files transitively. Track visited absolute paths to prevent infinite loops from circular references.

- **Cached remote modules** (`.terraform/modules/` directory present in the project root): include all `.tf` files found inside it. These are locally downloaded copies and must be scanned identically to source files.

- **Unresolved remote sources** (registry addresses, git URLs, HTTP URLs — not present on disk): record the source string in a "Referenced remote modules — not scanned" list for the report. Do not attempt to fetch them.

#### 2c. Build the final file list

Deduplicate all discovered paths. Record counts by origin:
- Root project files
- Local module files
- Cached remote module files (`.terraform/modules/`)

This combined list is the complete scan scope for step 3.

### 3. Analyse for security issues

Read every file in the scan scope. For each finding record:

| Field | Description |
|---|---|
| **ID** | Sequential: `TF-001`, `TF-002`, … |
| **Severity** | `CRITICAL` or `HIGH` |
| **Category** | One of the categories below |
| **Rule** | Short rule name, e.g. `OPEN_SSH_INGRESS` |
| **File** | Path relative to the selected project root |
| **Line** | Line number(s) |
| **Resource** | The Terraform resource name/label where applicable |
| **Description** | What is wrong and why it matters |
| **Recommendation** | The exact change needed to remediate it |

#### Security categories

**IAM & Permissions**
- Wildcard actions (`"Action": "*"`) or wildcard resources (`"Resource": "*"`) in IAM policies without compensating conditions
- `assume_role_policy` allowing any principal (`"AWS": "*"` or `"*"`)
- IAM inline policies on users, groups, or roles (prefer managed policies)
- Lambda execution roles with `AdministratorAccess` or equivalent wildcard policies
- Missing MFA condition on IAM roles used by humans
- `iam_role` with overly broad trust relationship (e.g. any service in any account)

**Secrets & Credentials**
- Hardcoded passwords, tokens, API keys, or private keys in any `.tf` or `.tfvars` file
- Variables with names suggesting secrets (`password`, `secret`, `token`, `key`, `api_key`) where `sensitive = false` or `sensitive` is absent
- Secrets passed as plaintext `environment` variables to Lambda, ECS task definitions, or EKS pods
- `aws_secretsmanager_secret_version` with `secret_string` set to a literal value rather than a reference

**Network & Public Exposure**
- Security group ingress rules with `cidr_blocks = ["0.0.0.0/0"]` or `ipv6_cidr_blocks = ["::/0"]` on ports: 22 (SSH), 3389 (RDP), 5432 (PostgreSQL), 3306 (MySQL), 1433 (MSSQL), 6379 (Redis), 27017 (MongoDB), 9200 (Elasticsearch), 8080, 8443
- S3 bucket `acl` set to `"public-read"` or `"public-read-write"`
- S3 `aws_s3_bucket_public_access_block` with any of `block_public_acls`, `block_public_policy`, `ignore_public_acls`, or `restrict_public_buckets` set to `false`
- `aws_s3_bucket` with no associated `aws_s3_bucket_public_access_block` resource
- RDS instance or cluster with `publicly_accessible = true`
- ALB or API Gateway stage using HTTP (port 80, no redirect to HTTPS)
- CloudFront distribution `viewer_protocol_policy` set to `"allow-all"` (permits plain HTTP)

**Encryption at Rest**
- S3 bucket with no `aws_s3_bucket_server_side_encryption_configuration`
- RDS instance or cluster with `storage_encrypted = false` or `storage_encrypted` absent
- EBS volume with `encrypted = false` or `encrypted` absent
- EBS snapshot with `encrypted = false`
- SQS queue with no `kms_master_key_id`
- SNS topic with no `kms_master_key_id`
- DynamoDB table with `server_side_encryption` block absent or `enabled = false`
- ElastiCache replication group with `at_rest_encryption_enabled = false`
- KMS key with `enable_key_rotation = false` or rotation absent

**Encryption in Transit**
- ElastiCache replication group with `transit_encryption_enabled = false`
- MSK cluster with `encryption_in_transit` client broker set to `"PLAINTEXT"`
- RDS with `iam_database_authentication_enabled = false` (absence of IAM auth increases plaintext credential risk)

**Logging & Auditing**
- S3 bucket with no `aws_s3_bucket_logging` resource
- CloudTrail with `enable_log_file_validation = false`
- CloudTrail that is not multi-region (`is_multi_region_trail = false` or absent)
- VPC with no associated `aws_flow_log` resource
- API Gateway stage with no `access_log_settings`
- ALB with `access_logs` block absent or `enabled = false`

**Container & Serverless**
- ECR repository with `image_scanning_configuration` absent or `scan_on_push = false`
- ECR repository with `image_tag_mutability = "MUTABLE"`
- Lambda function with `tracing_config` absent or `mode = "PassThrough"`

**Resilience & Safety**
- RDS instance or cluster with `deletion_protection = false` or absent
- DynamoDB table with `point_in_time_recovery` absent or `enabled = false`
- Terraform backend (`terraform { backend {} }`) with no encryption or state locking configured (e.g. S3 backend missing `encrypt = true` or DynamoDB lock table)

**WAF & DDoS**
- `aws_api_gateway_stage` or `aws_cloudfront_distribution` with no associated `aws_wafv2_web_acl_association` or `web_acl_id`

### 4. Filter and sort findings

Retain only `CRITICAL` and `HIGH` severity findings. If none exist, the report must clearly state **"No HIGH or CRITICAL issues found"** — do not omit the section.

Sort order: `CRITICAL` before `HIGH`; within the same severity, sort alphabetically by file path, then by line number.

### 5. Generate the HTML report

**Filename:** take the last two path components of the selected directory, join with `-`, append severity stamp and timestamp:
```
<slug>-security-<YYYY-MM-DD>-<HHmm>.html
```
Example: `my-infra/terraform/prod` at 14:32 → `terraform-prod-security-2026-05-19-1432.html`

**Save to:**
```
~/development/infra-ai-workshop/reports/terraform-security/<filename>.html
```
Create the directory if it does not exist.

#### Design

- **Dark theme**: `#0f172a` background, `#0f172a`→`#1e3a5f` gradient header
- **Google Fonts**: `Inter` body, `JetBrains Mono` for code/paths (via `@import`)
- **Severity colours**:
  - `CRITICAL` badge: `#ef4444` background, white text
  - `HIGH` badge: `#f97316` background, white text
- All CSS inline in a `<style>` block — no external stylesheets beyond the Google Fonts `@import`
- No external JavaScript libraries

#### HTML structure

```
<html>
  <head>  <!-- @import fonts, inline <style> -->
  <body>

    <!-- Sticky nav bar -->
    <nav class="sticky-bar">
      CRITICAL: N  |  HIGH: N  |  Files scanned: N

    <header>
      <!-- gradient banner -->
      <!-- Project path | Review date | Files scanned breakdown -->
      <!-- Modules scanned list -->
      <!-- Unresolved remote modules (not scanned) list — if any -->

    <section class="executive-summary">
      <!-- Overall risk level pill: CRITICAL / HIGH / PASS -->
      <!-- One-paragraph plain-English summary of the security posture -->

    <section class="findings-summary">
      <!-- Table: ID | Severity | Category | Rule | File | Line -->
      <!-- Each ID is an anchor link to the full finding card below -->

    <section class="findings-detail">
      <!-- For each finding: -->
      <div class="finding-card severity-{critical|high}" id="TF-001">
        <!-- Severity badge | ID | Category | Rule name -->
        <!-- Resource: <terraform resource label> -->
        <!-- File: <path>  Line: <N> (monospace pill) -->
        <!-- Description paragraph -->
        <!-- Recommendation (styled as a blue-tinted callout box) -->

    <footer>
      <!-- Generated by Claude Code | <timestamp> | AWS account (if available) -->
      <!-- Unresolved remote modules list -->
      <!-- Skipped/unreadable files list -->
  </body>
</html>
```

#### Visual details

- Finding cards: 4px left border in severity colour
- CRITICAL cards: `rgba(239,68,68,0.05)` background tint
- HIGH cards: `rgba(249,115,22,0.05)` background tint
- File/line references: monospace font, dark pill (`#1e293b` background, `#94a3b8` text)
- Recommendation callout: `rgba(59,130,246,0.08)` background, `#3b82f6` left border
- Findings summary table: zebra-striped rows, sticky header, sortable by severity
- Category headings in the detail section group findings visually

### 6. Open the report

```
open ~/development/infra-ai-workshop/reports/terraform-security/<filename>.html
```

---

## Constraints

- **READ-ONLY**: No file writes other than the HTML report output. No `terraform` commands of any kind.
- Static analysis only — do not execute, plan, init, validate, or apply any Terraform configuration.
- Do not fetch or download remote module sources. Only scan files already present on disk.
- Follow local module references transitively; track visited absolute paths to prevent infinite loops.
- If a file is binary or cannot be read, skip it and list it in the report footer under "Skipped files".
- Never display `null`, `undefined`, `NaN`, or raw error objects in the HTML output.
- All rendering must use inline HTML/CSS only — no external JS libraries.
