# Terraform Security Review Prompt

## CRITICAL CONSTRAINT — READ-ONLY

**No write, create, update, or delete operations are permitted under any circumstances.**
All file and AWS API access must be strictly read-only. Do not modify any Terraform files, state files, AWS resources, or any other content. Only read files and generate the local HTML report output.

---

## Task

Perform a security review of a Terraform codebase located within `~/development`. Identify all **HIGH** and **CRITICAL** severity findings and produce a self-contained HTML report saved under `~/development/infra-ai-workshop/reports/terraform-security/`.

---

## Steps

### 1. Discover and present available Terraform projects

Run the following command to find all directories under `~/development` that contain at least one `.tf` file:

```
find ~/development -name "*.tf" -not -path "*/.terraform/*" -type f | sed 's|/[^/]*\.tf$||' | sort -u
```

Take the resulting list of directories and render them as an indented tree, grouped by their path hierarchy. For example:

```
~/development/
├── my-infra/
│   ├── terraform/
│   │   ├── prod/
│   │   └── staging/
│   └── modules/
│       └── networking/
└── another-repo/
    └── infra/
```

Present this tree to the user and ask:

> Here are the Terraform projects found under `~/development`. Which one would you like to review? You can type the full path or the number next to it.

Number each leaf directory in the list for easy selection. Wait for the user to choose before proceeding.

Resolve the chosen path to its full absolute form. Confirm the directory exists before proceeding. If it does not exist, inform the user and stop.

### 2. Discover Terraform files and resolve modules (read-only)

#### 2a. Find all `.tf` and `.tfvars` files in the selected project

```
find <selected-path> -name "*.tf" -o -name "*.tfvars" -o -name "*.tfvars.json" | grep -v "/.terraform/" | sort
```

#### 2b. Discover referenced modules

Read every `.tf` file found above and extract all `module` blocks. For each `source` attribute:

- **Local path** (`source` starts with `./` or `../`): resolve the path relative to the `.tf` file that declares it. Add all `.tf` files found recursively under that resolved directory to the scan scope. Repeat this step for any new module references found inside those files (i.e. follow the chain transitively until no new paths are found).

- **`.terraform/modules/` cache** (remote modules already initialised locally): if a `.terraform/modules/` directory exists within the selected project, include all `.tf` files found inside it. These are downloaded copies of remote modules (registry, git) and must be scanned in the same way as local source files.

- **Unresolved remote sources** (registry paths, git URLs — not downloaded): record them in a "Referenced remote modules (not scanned)" list for the report footer. Do not attempt to download or fetch them.

#### 2c. Build the final file list

Collect all unique `.tf`, `.tfvars`, and `.tfvars.json` file paths across:
- The selected project directory (step 2a)
- All resolved local module directories (step 2b)
- `.terraform/modules/` cached remote modules (step 2b)

Deduplicate the list. This is the complete set of files to analyse in step 3.

Note in the report header: total files scanned, how many came from the root project vs local modules vs cached remote modules.

### 3. Analyse for security issues

Read every discovered `.tf` and `.tfvars` file and evaluate them against the following categories. For each finding, determine its **severity** (`CRITICAL` or `HIGH`) and record:

- **ID** — sequential finding number (e.g. `TF-001`)
- **Severity** — `CRITICAL` or `HIGH`
- **File** — relative file path from the project root
- **Line** — line number(s) where the issue appears
- **Rule** — short rule name (see categories below)
- **Description** — what is wrong and why it matters
- **Recommendation** — the specific change needed to remediate it

#### Security Categories to Check

**Identity & Access Management**
- Overly permissive IAM policies (`"*"` actions or resources without conditions)
- `assume_role_policy` that allows any principal (`"AWS": "*"`)
- IAM inline policies instead of managed policies
- Missing MFA conditions on sensitive roles

**Secrets & Credentials**
- Hardcoded secrets, passwords, tokens, or API keys in any `.tf` or `.tfvars` file
- `sensitive = false` on variables that clearly contain secrets
- Secrets passed as plaintext environment variables to Lambda or ECS

**Network & Exposure**
- Security groups with `0.0.0.0/0` or `::/0` ingress on sensitive ports (22, 3389, 5432, 3306, 6379, 27017)
- S3 buckets with `acl = "public-read"` or `acl = "public-read-write"`
- `block_public_acls`, `block_public_policy`, `ignore_public_acls`, or `restrict_public_buckets` set to `false` on S3 buckets
- RDS instances with `publicly_accessible = true`
- Load balancers or API Gateways using HTTP (not HTTPS)

**Encryption**
- S3 buckets without `server_side_encryption_configuration`
- RDS instances with `storage_encrypted = false`
- EBS volumes with `encrypted = false`
- SQS queues without KMS encryption
- SNS topics without KMS encryption

**Logging & Auditing**
- S3 buckets without access logging enabled
- CloudTrail with `enable_log_file_validation = false` or `is_multi_region_trail = false`
- VPC Flow Logs not enabled on VPCs
- API Gateway without access logging

**Miscellaneous**
- `deletion_protection = false` on RDS or DynamoDB
- Terraform state backend without encryption or locking configured
- Use of deprecated or insecure TLS versions

### 4. Filter to HIGH and CRITICAL only

Only include findings with severity `HIGH` or `CRITICAL` in the report. If no such findings exist, the report should clearly state "No HIGH or CRITICAL issues found."

### 5. Generate the HTML report

Determine the report filename from the target path:
- Take the last two path components of the target directory, joined with a dash
- Append a datestamp and timestamp: `<project-slug>-security-<YYYY-MM-DD>-<HHmm>.html`
- `<HHmm>` is the current local time in 24-hour format (e.g. `1432`), so multiple runs on the same day do not overwrite each other
- Example: for path `my-infra/terraform/prod` at 14:32 → `terraform-prod-security-2026-05-19-1432.html`

Save to:
```
~/development/infra-ai-workshop/reports/terraform-security/<filename>.html
```

Create the directory if it does not exist.

#### Report Design

- **Dark theme**, consistent with cost report styling (`#0f172a` background, `#1e3a5f` header gradient)
- **Fully self-contained** — all CSS inline in a `<style>` block, no external JS or stylesheets (Google Fonts via `@import` is acceptable)
- Severity badges:
  - `CRITICAL` → red (`#ef4444`) with white text
  - `HIGH` → orange (`#f97316`) with white text

#### Report Structure

```
<header>
  Project path reviewed
  Total files scanned (root: N | local modules: N | cached remote modules: N)
  Modules scanned: list of resolved module paths
  Unresolved remote modules: list (not scanned)
  Review date
  Finding counts: CRITICAL (N) | HIGH (N)

<section class="executive-summary">
  One-paragraph plain-English summary of the overall security posture
  Risk level indicator: CRITICAL / HIGH / MEDIUM / LOW / PASS

<section class="findings">
  For each finding (CRITICAL first, then HIGH, sorted by file):
    <div class="finding-card severity-{critical|high}">
      Severity badge | ID | Rule name
      File: <path>  Line: <N>
      Description paragraph
      Recommendation paragraph (styled as a callout box)

<footer>
  Generated by Claude Code | Timestamp
```

#### Visual Details

- Finding cards have a left border coloured by severity (4px solid)
- CRITICAL findings have a subtle red background tint (`rgba(239,68,68,0.05)`)
- HIGH findings have a subtle orange background tint (`rgba(249,115,22,0.05)`)
- A sticky top nav bar shows `CRITICAL: N  |  HIGH: N` at a glance
- Code snippets (file paths, line references) use a monospace font with a dark pill background

### 6. Open the report

```
open ~/development/infra-ai-workshop/reports/terraform-security/<filename>.html
```

---

## Constraints

- **READ-ONLY**: No file writes other than the HTML report output. No Terraform plan, apply, init, or validate commands that contact remote APIs or modify local state.
- Do not execute any Terraform commands — analyse the source files statically.
- Do not attempt to download or fetch remote module sources. Only scan modules that are already present on disk (local relative paths or `.terraform/modules/` cache).
- Follow local module references transitively but guard against circular references — track visited paths and stop if a path has already been scanned.
- If a file is binary or unreadable, skip it and note it in the report footer.
- All chart and badge rendering must use inline HTML/CSS — no external JS libraries.
