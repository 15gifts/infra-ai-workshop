# Terraform Security Review Prompt

## CRITICAL CONSTRAINT — READ-ONLY

**No write, create, update, or delete operations are permitted under any circumstances.**
All file access must be strictly read-only. Do not modify any Terraform files, state files, AWS resources, or any other content. The only permitted write is the HTML report output file.

## MCP server

This project uses the **`aws-mcp`** MCP server. If any AWS API calls are needed during the review (e.g. verifying resource existence), they must be executed via the `aws-mcp` `call_aws` MCP tool — not the Bash shell. The terraform review is primarily static file analysis and does not require AWS credentials, but the MCP server must be available if AWS lookups become necessary.

---

## Task

Perform a static security review of a Terraform codebase located within `~/development`. Identify all **HIGH** and **CRITICAL** severity findings and produce a self-contained HTML report saved under `~/development/infra-ai-workshop/reports/terraform-security/`.

---

## Steps

### 1. Ask for the scan root and discover available Terraform projects

First, ask the user:

> Which path would you like to scan for Terraform projects? Press Enter to use the default (`~/development`).

If the user provides a path, resolve it to an absolute path and confirm it exists before continuing. If it does not exist, tell the user and ask again. If the user provides no input (empty response), use `~/development` as the scan root.

Refer to the chosen path as `<SCAN_ROOT>` for the rest of this step.

Run the following command to find all directories under `<SCAN_ROOT>` that contain at least one `.tf` file, excluding cache directories:

```
find <SCAN_ROOT> \( -name "*.tf" \) \
  -not -path "*/.terraform/*" \
  -not -path "*/.terragrunt-cache/*" \
  -type f \
  | sed 's|/[^/]*\.tf$||' | sort -u
```

Note: on large trees this may take a few seconds. If the result set is very large, inform the user and offer to let them narrow the path.

If no results are returned, inform the user that no Terraform projects were found under `<SCAN_ROOT>` and stop.

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

Extract all `source` values from `module` blocks across the root project files:

```
grep -rh 'source\s*=' <selected-path> --include="*.tf" | grep -v '#' \
  | sed 's/.*source\s*=\s*["\x27]\(.*\)["\x27].*/\1/'
```

For each extracted source value:

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
| **AVD Reference** | Aqua AVD URL for this rule (see per-rule URLs in categories below) |
| **File** | Path relative to the selected project root |
| **Line** | Line number(s) |
| **Resource** | The Terraform resource name/label where applicable |
| **Description** | What is wrong and why it matters |
| **Recommendation** | The exact change needed to remediate it |

#### Severity definitions

Assign severity based on the following guidance — do not deviate from these assignments:

- **CRITICAL**: direct path to data exposure, account compromise, or loss of infrastructure control. Requires no additional exploitation step.
- **HIGH**: weakens the security posture materially, but typically requires an additional step or condition to be exploited (e.g. an attacker already has network access, or a misconfigured service must be discovered first).

#### Security categories and severity assignments

Each rule below includes its AVD reference URL. Use this URL verbatim as the `AVD Reference` field for any matching finding. For rules marked with a category-level URL, use that URL — no specific AVD ID exists for that check.

**IAM & Permissions**
- `CRITICAL` — Wildcard actions (`"Action": "*"`) or wildcard resources (`"Resource": "*"`) in IAM policies without compensating conditions → https://avd.aquasec.com/misconfig/aws/iam/avd-aws-0057/
- `CRITICAL` — `assume_role_policy` allowing any principal (`"AWS": "*"` or `"*"`) → https://avd.aquasec.com/misconfig/aws/iam/avd-aws-0056/
- `CRITICAL` — Lambda execution roles with `AdministratorAccess` or equivalent wildcard policies → https://avd.aquasec.com/misconfig/aws/iam/avd-aws-0057/
- `CRITICAL` — Terraform state backend with no encryption (`encrypt = false` or absent on S3 backend) → https://avd.aquasec.com/misconfig/aws/
- `HIGH` — IAM inline policies on users, groups, or roles → https://avd.aquasec.com/misconfig/aws/iam/avd-aws-0050/
- `HIGH` — Missing MFA condition on IAM roles used by humans → https://avd.aquasec.com/misconfig/aws/iam/
- `HIGH` — `iam_role` with overly broad trust relationship → https://avd.aquasec.com/misconfig/aws/iam/

**Secrets & Credentials**
- `CRITICAL` — Hardcoded passwords, tokens, API keys, or private keys in any `.tf` or `.tfvars` file → https://avd.aquasec.com/misconfig/aws/
- `CRITICAL` — `aws_secretsmanager_secret_version` with `secret_string` set to a literal value → https://avd.aquasec.com/misconfig/aws/secretsmanager/
- `HIGH` — Variables suggesting secrets (`password`, `secret`, `token`, `key`, `api_key`) where `sensitive = true` is absent → https://avd.aquasec.com/misconfig/aws/
- `HIGH` — Secrets passed as plaintext `environment` variables to Lambda, ECS, or EKS → https://avd.aquasec.com/misconfig/aws/

**Network & Public Exposure**
- `CRITICAL` — Security group ingress open to `0.0.0.0/0` or `::/0` on port 22 (SSH) → https://avd.aquasec.com/misconfig/aws/ec2/avd-aws-0107/
- `CRITICAL` — Security group ingress open to `0.0.0.0/0` or `::/0` on port 3389 (RDP) → https://avd.aquasec.com/misconfig/aws/ec2/avd-aws-0108/
- `CRITICAL` — Security group ingress open to `0.0.0.0/0` or `::/0` on other sensitive ports (5432, 3306, 1433, 6379, 27017, 9200) → https://avd.aquasec.com/misconfig/aws/ec2/avd-aws-0107/
- `CRITICAL` — S3 bucket `acl` set to `"public-read"` or `"public-read-write"` → https://avd.aquasec.com/misconfig/aws/s3/avd-aws-0090/
- `CRITICAL` — S3 `aws_s3_bucket_public_access_block` with any block attribute set to `false` → https://avd.aquasec.com/misconfig/aws/s3/avd-aws-0092/
- `CRITICAL` — S3 bucket policy allowing `"Principal": "*"` without a condition → https://avd.aquasec.com/misconfig/aws/s3/avd-aws-0090/
- `CRITICAL` — RDS instance or cluster with `publicly_accessible = true` → https://avd.aquasec.com/misconfig/aws/rds/avd-aws-0077/
- `HIGH` — `aws_s3_bucket` with no associated `aws_s3_bucket_public_access_block` resource → https://avd.aquasec.com/misconfig/aws/s3/avd-aws-0092/
- `HIGH` — ALB or API Gateway stage serving HTTP without a redirect to HTTPS → https://avd.aquasec.com/misconfig/aws/elb/
- `HIGH` — CloudFront `viewer_protocol_policy` set to `"allow-all"` → https://avd.aquasec.com/misconfig/aws/cloudfront/

**Encryption at Rest**
- `HIGH` — S3 bucket with no `aws_s3_bucket_server_side_encryption_configuration` → https://avd.aquasec.com/misconfig/aws/s3/avd-aws-0088/
- `HIGH` — RDS instance or cluster with `storage_encrypted = false` or absent → https://avd.aquasec.com/misconfig/aws/rds/avd-aws-0079/
- `HIGH` — EBS volume with `encrypted = false` or absent → https://avd.aquasec.com/misconfig/aws/ec2/avd-aws-0131/
- `HIGH` — EBS snapshot with `encrypted = false` → https://avd.aquasec.com/misconfig/aws/ec2/avd-aws-0131/
- `HIGH` — SQS queue with no `kms_master_key_id` → https://avd.aquasec.com/misconfig/aws/sqs/
- `HIGH` — SNS topic with no `kms_master_key_id` → https://avd.aquasec.com/misconfig/aws/sns/
- `HIGH` — DynamoDB table with `server_side_encryption` absent or `enabled = false` → https://avd.aquasec.com/misconfig/aws/dynamodb/avd-aws-0024/
- `HIGH` — ElastiCache replication group with `at_rest_encryption_enabled = false` → https://avd.aquasec.com/misconfig/aws/elasticache/
- `HIGH` — KMS key with `enable_key_rotation = false` or absent → https://avd.aquasec.com/misconfig/aws/kms/avd-aws-0065/

**Encryption in Transit**
- `HIGH` — ElastiCache replication group with `transit_encryption_enabled = false` → https://avd.aquasec.com/misconfig/aws/elasticache/
- `HIGH` — MSK cluster with `encryption_in_transit` client broker set to `"PLAINTEXT"` → https://avd.aquasec.com/misconfig/aws/msk/
- `HIGH` — RDS instance with `iam_database_authentication_enabled = false` → https://avd.aquasec.com/misconfig/aws/rds/

**Logging & Auditing**
- `HIGH` — S3 bucket with no `aws_s3_bucket_logging` resource → https://avd.aquasec.com/misconfig/aws/s3/avd-aws-0089/
- `HIGH` — CloudTrail with `enable_log_file_validation = false` → https://avd.aquasec.com/misconfig/aws/cloudtrail/avd-aws-0015/
- `HIGH` — CloudTrail not multi-region (`is_multi_region_trail = false` or absent) → https://avd.aquasec.com/misconfig/aws/cloudtrail/avd-aws-0014/
- `HIGH` — VPC with no associated `aws_flow_log` resource → https://avd.aquasec.com/misconfig/aws/ec2/avd-aws-0178/
- `HIGH` — API Gateway stage with no `access_log_settings` → https://avd.aquasec.com/misconfig/aws/apigateway/
- `HIGH` — ALB with `access_logs` block absent or `enabled = false` → https://avd.aquasec.com/misconfig/aws/elb/

**Container & Serverless**
- `HIGH` — ECR repository with `image_scanning_configuration` absent or `scan_on_push = false` → https://avd.aquasec.com/misconfig/aws/ecr/avd-aws-0030/
- `HIGH` — ECR repository with `image_tag_mutability = "MUTABLE"` → https://avd.aquasec.com/misconfig/aws/ecr/avd-aws-0031/
- `HIGH` — Lambda function with `tracing_config` absent or `mode = "PassThrough"` → https://avd.aquasec.com/misconfig/aws/lambda/

**Resilience & Safety**
- `HIGH` — RDS instance or cluster with `deletion_protection = false` or absent → https://avd.aquasec.com/misconfig/aws/rds/avd-aws-0080/
- `HIGH` — DynamoDB table with `point_in_time_recovery` absent or `enabled = false` → https://avd.aquasec.com/misconfig/aws/dynamodb/
- `HIGH` — Terraform state backend with no DynamoDB lock table configured → https://avd.aquasec.com/misconfig/aws/

**WAF & DDoS**
- `HIGH` — `aws_api_gateway_stage` or `aws_cloudfront_distribution` with no associated WAF → https://avd.aquasec.com/misconfig/aws/

If a file cannot be parsed as valid HCL (syntax error or unexpected encoding), skip module resolution and security checks for that file and record it under "Skipped files — parse error" in the report footer.

### 4. Filter and sort findings

Retain only `CRITICAL` and `HIGH` severity findings. If none exist, the report must clearly state **"No HIGH or CRITICAL issues found"** — do not omit the section.

Sort order: `CRITICAL` before `HIGH`; within the same severity, sort alphabetically by file path, then by line number.

### 5. Generate the HTML report

**Filename:** take the last two path components of the selected directory (relative to `~/development`), join with `-`, append a date and timestamp:
```
<slug>-security-<YYYY-MM-DD>-<HHmm>.html
```
- If the path has only one component (e.g. `~/development/infra`), use the parent directory name as the first component: `development-infra-security-...html`
- Example: `my-infra/terraform/prod` at 14:32 → `terraform-prod-security-2026-05-19-1432.html`

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
      <!-- Table: ID | Severity | Category | Rule | File | Line | Reference -->
      <!-- Each ID is an anchor link to the full finding card below -->
      <!-- Reference column: "AVD ↗" as a small external link to the AVD URL -->

    <section class="findings-detail">
      <!-- For each finding: -->
      <div class="finding-card severity-{critical|high}" id="TF-001">
        <!-- Severity badge | ID | Category | Rule name -->
        <!-- Resource: <terraform resource label> -->
        <!-- File: <path>  Line: <N> (monospace pill) -->
        <!-- Description paragraph -->
        <!-- Recommendation (styled as a blue-tinted callout box) -->
        <!-- Reference: "AVD Reference: <avd-url>" as a clickable link, opening in a new tab -->

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
- Findings summary table: zebra-striped rows, sticky header — include a small inline `<script>` block (no external libraries) that adds click-to-sort on column headers. Sort by Severity by default (CRITICAL first).
- Category headings group findings in the detail section visually; within each category, CRITICAL cards appear before HIGH

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
- If a file is binary, unparseable, or unreadable, skip it and list it in the report footer under "Skipped files".
- Never display `null`, `undefined`, `NaN`, or raw error objects in the HTML output.
- Inline `<script>` blocks for table interactivity are permitted. External JS libraries are not.
