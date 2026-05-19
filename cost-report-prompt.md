# AWS Cost Report Prompt

## CRITICAL CONSTRAINT — READ-ONLY

**No write, create, update, or delete operations are permitted under any circumstances.**
All AWS API calls must be strictly read-only (GET/LIST/DESCRIBE operations). Do not create, modify, or delete any AWS resources, IAM policies, S3 objects, Cost Explorer saved reports, or any other AWS-managed state. If a step seems to require a write operation, skip it and note the limitation instead.

---

## Task

Generate a polished HTML cost report from AWS Cost Explorer for the **production environment**, broken down by region and team. The account has production infrastructure in two regions:

- **Production US** — `us-east-1`
- **Production EU** — `eu-west-1`

Resources are tagged with `Product` or `SubProduct` (values: `platform`, `humara`, `axiom`) to identify team ownership, and with `Environment` (`live`, `prod`, or `production`) to identify production resources. The report covers the **last two full calendar months** (for month-over-month comparison) and is saved under `~/development/infra-ai-workshop/reports/cost-report/`.

---

## Steps

### 1. Verify AWS access

```
aws sts get-caller-identity
```

If this fails, inform the user that AWS credentials are not configured or are expired and stop. Store the `Account` value for use in the report footer.

### 2. Ask the user which region to report on

Present the following choice and wait for a response before proceeding:

> Which region would you like to report on?
> 1. **Production US** — us-east-1
> 2. **Production EU** — eu-west-1
> 3. **Both** — combined report with regional comparison

Store the selection as `SELECTED_REGIONS`:
- Option 1 → `["us-east-1"]`
- Option 2 → `["eu-west-1"]`
- Option 3 → `["us-east-1", "eu-west-1"]`

### 3. Determine the date range

Use today's date to compute:
- `PREV_START`: first day of the month before last (e.g. `2026-03-01`)
- `PREV_END` / `CURR_START`: first day of last month (e.g. `2026-04-01`)
- `CURR_END`: first day of the current month (e.g. `2026-05-01`)

The **primary reporting month** is `CURR_START`→`CURR_END`. The filename `<YYYY-MM>` token uses `CURR_START` formatted as `YYYY-MM`. The prior month is used only for MoM delta calculations.

### 4. Pull cost data

The CE API endpoint is always `us-east-1` regardless of which region's data is being requested. Data is scoped to a region using a `DIMENSION:REGION` filter.

For each region in `SELECTED_REGIONS`, for each tag key (`Product`, `SubProduct`), for each time period (`CURR`, `PREV`), run the following query — **up to 8 queries total** when both regions are selected:

```
aws ce get-cost-and-usage \
  --region us-east-1 \
  --time-period Start=<START>,End=<END> \
  --granularity MONTHLY \
  --metrics "BlendedCost" "UnblendedCost" \
  --group-by Type=TAG,Key=<TAG_KEY> Type=DIMENSION,Key=SERVICE \
  --filter '{
    "And": [
      {
        "Dimensions": {
          "Key": "REGION",
          "Values": ["<REGION>"],
          "MatchOptions": ["EQUALS"]
        }
      },
      {
        "Tags": {
          "Key": "Environment",
          "Values": ["live", "prod", "production"],
          "MatchOptions": ["EQUALS"]
        }
      },
      {
        "Or": [
          {
            "Tags": {
              "Key": "<TAG_KEY>",
              "Values": ["platform", "humara", "axiom"],
              "MatchOptions": ["EQUALS"]
            }
          },
          {
            "Tags": {
              "Key": "<TAG_KEY>",
              "MatchOptions": ["ABSENT"]
            }
          }
        ]
      }
    ]
  }'
```

If the API returns an error (e.g. Cost Explorer not enabled, insufficient permissions), display the error message and stop.

**Parsing the response:** Each item in `ResultsByTime[0].Groups` has `Keys: [tag_value, service_name]` and `Metrics.BlendedCost.Amount`.

**Merging tag key results:** For each `(region, team, service)` combination, sum `BlendedCost` from both the `Product` and `SubProduct` queries. Resources are expected to carry one tag key or the other — if a resource carries both tags with the same value it may be double-counted; note this caveat in the report footer.

**Untagged resources:** Rows where `tag_value == ""` represent production resources with no `Product`/`SubProduct` tag. Label these `Untagged`, include in totals, and style with a grey dashed border.

**Parsing the response:** Each item in `ResultsByTime[0].Groups` has `Keys: [tag_value, service_name]` and `Metrics.BlendedCost.Amount`. The `tag_value` is the raw tag value (e.g. `"platform"`, `"humara"`, `"axiom"`, or `""` for absent-tagged resources).

**Merging Product and SubProduct results:** For each team value, sum the `BlendedCost` from both the `Product` query and the `SubProduct` query. If a resource carries both tags set to the same value it may appear in both queries — note this caveat in the report footer. In practice, resources are expected to use one tag key or the other, not both.

**Untagged resources:** Rows where `tag_value == ""` represent production resources with no `Product`/`SubProduct` tag. Label these `Untagged` in the report and include them in the grand total with a grey dashed border.

### 5. Compute derived values

Organise data as `costs[region][team][service]` for both the current and prior months.

For **each region in `SELECTED_REGIONS`**, for **each team** (`platform`, `humara`, `axiom`, `Untagged`):
- **Team total (current month)**: sum `BlendedCost` across all services for that region + team
- **Team total (prior month)**: same from prior-month queries
- **MoM delta**: `((current - prior) / prior) × 100` — `+X.X%` green or `-X.X%` red. "New spend" if prior = $0; "—" if both = $0.
- **Top 5 services** by blended cost, current month
- **Each service as % of team total**

Also compute for **each region**:
- **Regional total (current)**: sum across all teams for that region
- **Regional total (prior)**: same
- **Regional MoM delta**

And for the **combined view** (when both regions selected):
- **Grand total (current)**: sum of all regional totals
- **Grand total (prior)**: sum of all regional totals for prior month
- **Grand total MoM delta** — shown in the summary banner; "First month of data" if prior = $0

### 6. Generate the HTML report

Create the output directory first:
```
mkdir -p ~/development/infra-ai-workshop/reports/cost-report
```

Save the report to:
```
~/development/infra-ai-workshop/reports/cost-report/cost-report-<YYYY-MM>-<HHmm>.html
```
where `<YYYY-MM>` = `CURR_START` formatted as `YYYY-MM` and `<HHmm>` = current local 24-hour time (e.g. `1432`).

#### Design

- **Dark theme**: `#0f172a` page background, `#0f172a`→`#1e3a5f` gradient header
- **Google Fonts**: `Inter` for body, `JetBrains Mono` for numbers/code (via `@import`)
- **Team colour coding**:
  - `platform` → indigo `#6366f1`
  - `humara` → emerald `#10b981`
  - `axiom` → amber `#f59e0b`
  - `Untagged` → slate `#64748b` (dashed border)
- All charts rendered as **inline SVG** — no external JS libraries

#### HTML structure

The layout adapts based on whether one or both regions were selected.

```
<html>
  <head>  <!-- @import fonts, inline <style> block -->
  <body>
    <header>
      <!-- gradient banner -->
      <!-- "Production Cost Report — <Month YYYY>" -->
      <!-- Scope: Production US / Production EU / Production US + EU -->
      <!-- AWS Account: <id> -->

    <!-- COMBINED VIEW ONLY (both regions selected) -->
    <section class="global-summary">
      <!-- Grand total across both regions: large centred figure + MoM delta badge -->
      <!-- Side-by-side regional totals: US $X,XXX.XX | EU $X,XXX.XX -->
      <!-- Donut chart (inline SVG): proportional spend by region (US vs EU) -->

    <!-- ONE SECTION PER SELECTED REGION -->
    <section class="region-block" id="us-east-1">  <!-- or eu-west-1 -->
      <!-- Region heading: "Production US — us-east-1" (or EU equivalent) -->
      <!-- Regional total + MoM delta -->
      <!-- Donut chart: spend per team within this region -->

      <div class="team-cards">
        <!-- One card per team: total cost, MoM delta, top service -->

      <div class="team-detail" id="<region>-platform">
      <div class="team-detail" id="<region>-humara">
      <div class="team-detail" id="<region>-axiom">
      <div class="team-detail" id="<region>-untagged">  <!-- only if spend > $0 -->
        <!-- Team name + colour accent bar -->
        <!-- Total blended cost (large) + MoM delta -->
        <!-- Top-5 services horizontal bar chart (inline SVG) -->
        <!-- Full service breakdown table: Service | Blended Cost | % of Team Total -->

    <footer>
      <!-- "Report generated: <datetime>" | "AWS Account: <id>" -->
      <!-- "Costs from Product and SubProduct tags are summed per team." -->
      <!-- "Resources carrying both tags with the same value may be counted twice." -->
  </body>
</html>
```

#### Inline SVG specifications

**Donut chart** — use `stroke-dasharray` on `<circle>` elements (simpler than arc paths):
- Outer circle radius: `70`, stroke-width: `28`, `fill="none"`, `cx="100" cy="100"`
- Circumference = `2 × π × 70 ≈ 439.8`
- For each team segment: `stroke-dasharray="<arc_length> <remaining>"` where `arc_length = team_fraction × 439.8`
- Rotate each segment using `stroke-dashoffset` so segments follow each other (offset = circumference × (1 − cumulative_fraction_before_this_team))
- Centred label: grand total as `$X,XXX.XX`
- Legend row below: colour swatch `■` + team name + `$X,XXX.XX`

**Horizontal bar chart** (per team, top 5 services):
- `viewBox="0 0 500 130"` (adjust height per bar count)
- Service label: left-aligned, monospace, max 28 chars with `…` truncation
- Bar: starts at x=200, max width 260px, height 18px, `rx="3"`
- Bar width = `(service_cost / team_total) × 260`
- Dollar label: right-aligned after bar end
- Bars filled with team accent colour at 80% opacity

#### Number formatting

- Currency: `$X,XXX.XX` (always 2 decimal places) — including the donut centre label
- MoM delta: `+X.X%` or `-X.X%` (1 decimal place)
- Percentages: `X.X%` (1 decimal place)
- Never display `NaN`, `Infinity`, `null`, or `undefined` — use `$0.00` or `—` as fallbacks
- If a team has $0 spend in the primary month, show "No spend recorded this month" placeholder instead of a chart

### 7. Open the report

```
open ~/development/infra-ai-workshop/reports/cost-report/cost-report-<YYYY-MM>-<HHmm>.html
```

---

## Constraints

- **READ-ONLY**: No AWS write, create, update, or delete API calls. Ever. Under any circumstances.
- Always query Cost Explorer with `--region us-east-1`
- All chart rendering must be **inline SVG** — no D3, Chart.js, or other external JS
- CSS must be fully inline in a `<style>` block — no external stylesheets beyond the Google Fonts `@import`
- The HTML file must be fully self-contained and renderable offline (except the font import)
