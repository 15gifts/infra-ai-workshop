# AWS Tag-Based Cost Report Prompt

## CRITICAL CONSTRAINT — READ-ONLY

**No write, create, update, or delete operations are permitted under any circumstances.**
All AWS API calls must be strictly read-only (GET/LIST/DESCRIBE operations). Do not create, modify, or delete any AWS resources, IAM policies, S3 objects, Cost Explorer saved reports, or any other AWS-managed state. If a step seems to require a write operation, skip it and note the limitation instead.

---

## Task

Generate a polished HTML cost report from AWS Cost Explorer, broken down by the tag key **`team`** with values **`platform`**, **`humara`**, and **`axiom`**. The report should cover the **last full calendar month** and be saved locally as `cost-report.html`.

---

## Steps

### 1. Determine the date range

Use today's date to compute:
- `START`: first day of the previous full month (e.g. `2026-04-01`)
- `END`: first day of the current month (e.g. `2026-05-01`)

### 2. Retrieve the AWS account identity (read-only)

```
aws sts get-caller-identity
```

Store the `Account` value for use in the report footer.

### 3. Pull cost data grouped by tag

Run a single query grouping by both `TAG:team` and `DIMENSION:SERVICE` to get per-team, per-service costs in one call:

```
aws ce get-cost-and-usage \
  --time-period Start=<START>,End=<END> \
  --granularity MONTHLY \
  --metrics "BlendedCost" "UnblendedCost" \
  --group-by Type=TAG,Key=team Type=DIMENSION,Key=SERVICE \
  --filter '{
    "Tags": {
      "Key": "team",
      "Values": ["platform", "humara", "axiom"],
      "MatchOptions": ["EQUALS"]
    }
  }'
```

This is a read-only Cost Explorer query — no resources are created or modified.

### 4. Generate `cost-report.html`

Write a self-contained HTML file to:
```
~/development/infra-ai-workshop/reports/cost-report/cost-report-<YYYY-MM>-<HHmm>.html
```
where `<YYYY-MM>` is the month covered by the report (e.g. `2026-04`) and `<HHmm>` is the current local time in 24-hour format (e.g. `1432`). This ensures multiple runs on the same day do not overwrite each other. Create the directory if it does not exist.

The file should have no external CDN dependencies except Google Fonts, with:

#### Layout & Design Requirements

- **Dark theme** with a gradient header (`#0f172a` → `#1e3a5f`)
- **Card-based layout** — one summary card per team tag, plus a full breakdown table
- **Color coding** by team:
  - `platform` → indigo (`#6366f1`)
  - `humara` → emerald (`#10b981`)
  - `axiom` → amber (`#f59e0b`)
- **Top summary bar** showing total spend across all three tags
- A **donut chart** (rendered via inline SVG — no external JS) showing proportional spend per team
- A **per-team section** for each tag value containing:
  - Total blended cost (large, prominent)
  - Top 5 services by cost as a horizontal bar chart (inline SVG)
  - Full service breakdown table: Service | Blended Cost | % of Team Total
- **Footer** with report generation timestamp and AWS account ID

#### HTML Structure

```
<html>
  <head> ... fonts, inline CSS ... </head>
  <body>
    <header>  <!-- gradient banner, report title, date range, account info -->
    <section class="summary">  <!-- total spend + donut chart -->
    <section class="team-cards">  <!-- 3 summary cards -->
    <section class="team-detail" id="platform"> ... </section>
    <section class="team-detail" id="humara">   ... </section>
    <section class="team-detail" id="axiom">    ... </section>
    <footer> ... </footer>
  </body>
</html>
```

#### Inline SVG Charts

- **Donut chart**: three arc segments proportional to each team's total cost, with a centered legend
- **Horizontal bars**: per team, top-5 services. Bar width = `(service_cost / team_total) * 100%`, dollar label on the right

#### Number Formatting

- Costs: `$X,XXX.XX`
- Percentages: one decimal place
- If a tag value has $0 spend, show a "No spend recorded" placeholder instead of a broken chart

### 5. Open the report

```
open ~/development/infra-ai-workshop/reports/cost-report/cost-report-<YYYY-MM>-<HHmm>.html
```

---

## Constraints

- **READ-ONLY**: No AWS write, create, update, or delete API calls. Ever. Under any circumstances.
- All chart rendering must be **inline SVG** — no D3, Chart.js, or other external JS libraries
- CSS must be **fully inline** in a `<style>` block — no external stylesheets
- The HTML file must be **fully self-contained** and renderable offline
- Round all currency values to **2 decimal places**
- Untagged resources returned under the `$` key should be labeled `Untagged`, included in totals, and visually distinguished with a grey dashed border
