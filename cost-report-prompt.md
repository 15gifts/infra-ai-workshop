# AWS Tag-Based Cost Report Prompt

## CRITICAL CONSTRAINT — READ-ONLY

**No write, create, update, or delete operations are permitted under any circumstances.**
All AWS API calls must be strictly read-only (GET/LIST/DESCRIBE operations). Do not create, modify, or delete any AWS resources, IAM policies, S3 objects, Cost Explorer saved reports, or any other AWS-managed state. If a step seems to require a write operation, skip it and note the limitation instead.

---

## Task

Generate a polished HTML cost report from AWS Cost Explorer, broken down by the tag key **`team`** with values **`platform`**, **`humara`**, and **`axiom`**. The report covers the **last two full calendar months** (to enable month-over-month comparison) and is saved under `~/development/infra-ai-workshop/reports/cost-report/`.

---

## Steps

### 1. Verify AWS access

Run:
```
aws sts get-caller-identity
```

If this fails, inform the user that AWS credentials are not configured or are expired and stop. Do not proceed. Store the `Account` value for the report footer.

### 2. Determine the date range

Use today's date to compute two full calendar months:
- `PREV_START`: first day of the month before last (e.g. `2026-03-01`)
- `PREV_END`: first day of last month (e.g. `2026-04-01`)
- `CURR_START`: first day of last month (e.g. `2026-04-01`)
- `CURR_END`: first day of the current month (e.g. `2026-05-01`)

The **primary reporting month** is `CURR_START`→`CURR_END`. The prior month (`PREV_START`→`PREV_END`) is used only for month-over-month delta calculations.

### 3. Pull cost data

Run two queries — one per month — each grouping by `TAG:team` and `DIMENSION:SERVICE`:

```
aws ce get-cost-and-usage \
  --time-period Start=<CURR_START>,End=<CURR_END> \
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

Repeat with `Start=<PREV_START>,End=<PREV_END>` for the prior month.

If the API returns an error (e.g. Cost Explorer not enabled, insufficient permissions), display the error message clearly and stop.

**Handling untagged resources:** AWS Cost Explorer returns costs with no matching tag under the key `""` (empty string). Label these as `Untagged` in the report, include them in the total, and style them distinctly (grey, dashed border).

### 4. Compute derived values

For each team tag value (`platform`, `humara`, `axiom`, and `Untagged` if present):
- **Total blended cost** for the primary month
- **Month-over-month delta**: `((current - prior) / prior) * 100` — show as `+X.X%` (green) or `-X.X%` (red). If prior month is $0, show "New spend" instead of a percentage.
- **Top 5 services** by blended cost for the primary month
- **Each service as a % of that team's total**

Also compute the **grand total** across all teams for the primary month.

### 5. Generate the HTML report

Save to:
```
~/development/infra-ai-workshop/reports/cost-report/cost-report-<YYYY-MM>-<HHmm>.html
```
where `<YYYY-MM>` is the primary reporting month (e.g. `2026-04`) and `<HHmm>` is the current local time in 24-hour format (e.g. `1432`). Create the directory if it does not exist.

#### Design

- **Dark theme**: `#0f172a` page background, `#0f172a`→`#1e3a5f` gradient header
- **Google Fonts**: `Inter` for body, `JetBrains Mono` for numbers/code (via `@import`)
- **Team colour coding**:
  - `platform` → indigo `#6366f1`
  - `humara` → emerald `#10b981`
  - `axiom` → amber `#f59e0b`
  - `Untagged` → slate `#64748b` (dashed border)
- All charts rendered as **inline SVG** — no external JS libraries

#### HTML Structure

```
<html>
  <head>  <!-- @import fonts, inline <style> block -->
  <body>
    <header>
      <!-- gradient banner -->
      <!-- report title, primary month, AWS account ID -->

    <section class="summary">
      <!-- grand total spend (large, centred) -->
      <!-- donut chart: proportional spend per team -->
      <!-- MoM total delta badge -->

    <section class="team-cards">
      <!-- one card per team: total cost, MoM delta arrow, top service -->

    <section class="team-detail" id="platform">
    <section class="team-detail" id="humara">
    <section class="team-detail" id="axiom">
    <section class="team-detail" id="untagged">  <!-- only if untagged spend > $0 -->
      <!-- each section: -->
      <!--   team name + colour accent -->
      <!--   total blended cost (large) + MoM delta -->
      <!--   top-5 services horizontal bar chart (inline SVG) -->
      <!--   full service breakdown table: Service | Blended Cost | % of Team Total -->

    <footer>
      <!-- "Report generated: <datetime>" | "AWS Account: <id>" -->
  </body>
</html>
```

#### Inline SVG specifications

**Donut chart** (viewBox `0 0 200 200`, centre `100,100`, radius `70`, hole radius `42`):
- One arc segment per team, proportional to share of grand total
- Each segment coloured with its team colour
- Centred label: grand total in `$X,XXX` format
- Legend below chart: colour swatch + team name + amount

**Horizontal bar chart** (per team, top 5 services):
- `viewBox` wide enough to fit labels; bars left-aligned
- Bar width = `(service_cost / team_total) * 300px` (max 300px)
- Service name label on the left (truncated at 24 chars with ellipsis if needed)
- Dollar amount label right-aligned after the bar
- Bars coloured with the team's accent colour at 80% opacity

#### Number formatting

- Currency: `$X,XXX.XX` (always 2 decimal places)
- MoM delta: `+X.X%` or `-X.X%` (1 decimal place)
- Percentages: `X.X%` (1 decimal place)
- If a team has $0 spend in the primary month, show a subtle "No spend recorded this month" placeholder — no broken charts

### 6. Open the report

```
open ~/development/infra-ai-workshop/reports/cost-report/cost-report-<YYYY-MM>-<HHmm>.html
```

---

## Constraints

- **READ-ONLY**: No AWS write, create, update, or delete API calls. Ever. Under any circumstances.
- All chart rendering must be **inline SVG** — no D3, Chart.js, or other external JS
- CSS must be fully inline in a `<style>` block — no external stylesheets beyond the Google Fonts `@import`
- The HTML file must be **fully self-contained** and renderable offline (except for the font import)
- Round all currency values to 2 decimal places; never display `NaN`, `Infinity`, or bare `null`
