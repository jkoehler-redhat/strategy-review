---
name: strategy-review
description: Generate a data-backed strategy review document for AICP leadership meetings — covers portfolio, deliverables, in-flight work, customer signal, and opportunities
---

# Strategy Review Skill

Use this skill when a manager asks for a strategy review, leadership deck prep, or invokes `/strategy-review`.

## What This Skill Does

Generates a comprehensive strategy review document from live Jira data, covering what AICP owns, what has been delivered, what is in flight, customer signal, and where opportunities exist. Output is a Word document and/or markdown file suitable for leadership meetings.

## When to Use

- Preparing for leadership reviews with Sherard Griffin or other directors
- Quarterly strategy planning sessions
- Portfolio health checks
- User asks for "strategy review", "leadership review", "portfolio review"
- User invokes `/strategy-review`

## Interactive Prompts

Before starting data collection, ask the user:

### Prompt 1: Lookback Period

```
Question: "How far back should the 'What We've Delivered' section look?"
Options:
  1. 90 days [default]
  2. 60 days
  3. 30 days
  4. Custom (specify number of days)
```

### Prompt 2: Output Format

```
Question: "What output format?"
Options:
  1. Word document (.docx) [default]
  2. Markdown (.md)
  3. Both
```

### Prompt 3: Additional Context

```
Question: "Any strategic context to include? (meeting outcomes, leadership asks, upcoming priorities)"
Note: Optional. The user can paste notes or skip entirely.
```

## Data Collection

### Source 1: AICP Feature Priorities Spreadsheet

The **primary source of truth** for what AICP owns vs supports is the Google Sheet:
`https://docs.google.com/spreadsheets/d/1CuOVl4JIjwrCTHmmbHoZepsrni74CstNwtFc_Rb_Nls`

Export the full spreadsheet (all tabs) via Drive API:

```bash
gws drive files export --params '{"fileId":"1CuOVl4JIjwrCTHmmbHoZepsrni74CstNwtFc_Rb_Nls","mimeType":"application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"}' -o aicp_feature_priorities.xlsx
```

Parse all tabs using Python `zipfile` + `xml.etree.ElementTree` (stdlib, no pip install needed).

#### Roadmap Tab — Key Columns

| Column | Field | Usage |
|--------|-------|-------|
| A | Feature/Initiative | Jira key (RHAISTRAT-*, RHOAIENG-*, RHAIRFE-*) |
| B | Title | Feature name |
| C | **Ownership** | `AICP` = team owns it, `Support` = reviewing/supporting another team's feature |
| D | Status | In Progress, New, Backlog, Refinement, Release Pending, Review |
| E | AICP Supporting Work | Linked RHOAIENG issues for support items |
| F | Planned Release | Target release (3.4, 3.5, etc.) |
| G | JIRA Target Version | Target version label |
| I | Target End Date | Deadline |
| J | Sub-Team | Forge, Compass, Heimdall, N/A |
| K | AICP Priority | Blocker, Critical, Major, Normal |
| L | AICP Effort | Small, Medium, Large, X-Large |
| M | Notes | Additional context |

**Critical distinction**:
- **Ownership = "AICP"**: Features this team owns and drives. These go in "What We Own" and "What's In Flight" as primary work.
- **Ownership = "Support"**: Features another team owns that AICP reviews, consults on, or does supporting engineering for. These go in a separate "Platform Support & Refinement Reviews" subsection.

Only use the **Roadmap** tab. The Refinements and Completed Refinements tabs are being deprecated — do not read them.

### Source 2: Jira REST API

### Authentication

Extract the Jira API token from the Claude Code configuration:

```
Read /Users/jaykoehler/.claude.json
Parse mcpServers.mcp-atlassian.args array
Find the --jira-token value (the element after "--jira-token")
Find the --jira-username value (the element after "--jira-username")
Store as JIRA_TOKEN and JIRA_EMAIL
```

### API Conventions

All queries use the Jira REST API v3 — **do NOT use MCP tools**.

```bash
curl -s -H "Authorization: Basic $(echo -n '{JIRA_EMAIL}:{JIRA_TOKEN}' | base64)" \
  -H "Content-Type: application/json" \
  "https://redhat.atlassian.net/rest/api/3/search/jql?jql={ENCODED_JQL}&fields={FIELDS}&maxResults=100"
```

**Pagination**: Use cursor-based pagination with `nextPageToken` / `isLast`. Do NOT rely on the `total` field — it returns -1 for complex queries. Count returned results manually.

**Component scope**: ALL queries MUST include `component in ("AI Core Platform", "AI Core Platform Security")`. This is non-negotiable — AI Core Platform IS this team. No Model Serving, Dashboard, MLflow, or other component issues should appear.

### Query 1: Portfolio Total

Count all open issues to establish portfolio size.

```
JQL: project = RHOAIENG
     AND component in ("AI Core Platform", "AI Core Platform Security")
     AND status NOT IN (Closed, Resolved, Done, Cancelled)
```

Paginate to get a complete count. Record total as the portfolio size.

### Query 2: In-Flight by Sub-Team

Active work categorized by sub-team label.

```
JQL: project = RHOAIENG
     AND component in ("AI Core Platform", "AI Core Platform Security")
     AND status in ("In Progress", "In Development", "In Review", "Code Review", "In Testing", "QE Review")
Fields: key, summary, status, assignee, labels, priority
```

Partition results by label:

| Label | Team |
|-------|------|
| `aicp-team-forge` | Forge — xKS Expansion, GitOps & CI/CD |
| `aicp-team-compass` | Compass — E2E Stability, QE & Observability |
| `aicp-team-heimdall` | Heimdall — Security, Gateway & Authentication |
| None of the above | Unassigned — flag for gap analysis |

### Query 3: Delivered (Lookback Period)

Resolved issues within the lookback period, grouped by theme.

```
JQL: project = RHOAIENG
     AND component in ("AI Core Platform", "AI Core Platform Security")
     AND resolved >= -{N}d
     AND status in (Closed, Resolved, Done, "Release Pending")
Fields: key, summary, status, assignee, labels, resolution, resolutiondate, priority
```

Where `{N}` is the lookback period from Prompt 1.

Partition by sub-team label (same as Query 2), then group by theme using keywords:

| Theme | Keywords |
|-------|----------|
| xKS / Cloud Controller Manager | xKS, helm, cloud controller, CCM, AKS, CoreWeave, cert-manager (xKS context), multi-cloud |
| Gateway & Authentication | BYOIDC, gateway, kube-auth-proxy, OAuth, OIDC, bearer token, auth, Entra ID |
| Observability | observability, metrics, monitoring, perses, tempo, telemetry, tracing, otel |
| Testing & QA | test, e2e, QE, tier1, tier2, tier3, sanity, sign-off, test plan, CI job, envtest |
| CI/CD & Infrastructure | CI, CD, prow, konflux, jenkins, pipeline, build, release |
| Operator Stability & Bug Fixes | operator, DSC, DSCI, reconcil, controller, webhook, informer, backport |
| Security | CVE, security, vulnerability, RBAC, cert, TLS, readOnly |
| Kueue / Batch Workloads | kueue, queue, jobset, batch, DRA |
| GitOps & Deployment | GitOps, kustomize, argocd, helm chart, deployment |
| Component Onboarding | "CLONE -", "Integration with ODH operator", "Integration with RHOAI operator" |

### Query 4: RHOAIENG Customer Signal

Customer-labeled issues within AICP scope.

```
JQL: project = RHOAIENG
     AND component in ("AI Core Platform", "AI Core Platform Security")
     AND labels in ("customer-bug", "AIBU_Feedback", "customer-escalation", "BU_EarlyAccess_Feedback", "AISSA_Feedback")
     AND status NOT IN (Closed, Resolved, Done)
Fields: key, summary, status, priority, assignee, labels
```

**Important**: AICP component rarely has direct customer-labeled issues. Customer pain typically surfaces on downstream components (Model Serving, Dashboard, etc.). Be honest about this in the report — zero results is normal and should be stated, not hidden.

### Query 5: RHAISTRAT Customer Signal

Strategic customer-driven features across the broader product.

```
JQL: project = RHAISTRAT
     AND (labels in ("field-request", "AIBU_Feedback", "telco", "BU_EarlyAccess_Feedback")
          OR summary ~ "customer")
     AND status NOT IN (Closed, Resolved, Done)
Fields: key, summary, status, priority, labels
```

Note: RHAISTRAT issues are NOT filtered by AI Core Platform component — they represent cross-product strategic items. Include them for context but clearly label them as RHAISTRAT scope.

### Query 6: Blockers Snapshot

Critical and blocker-priority issues currently open.

```
JQL: project = RHOAIENG
     AND component in ("AI Core Platform", "AI Core Platform Security")
     AND priority in (Blocker, Critical)
     AND status NOT IN (Closed, Resolved, Done, Cancelled)
Fields: key, summary, status, priority, assignee, labels
```

### Query 7: Unassigned Gap

In-flight issues without any sub-team label — represents capacity or ownership gaps.

```
JQL: project = RHOAIENG
     AND component in ("AI Core Platform", "AI Core Platform Security")
     AND status in ("In Progress", "In Development", "In Review", "Code Review", "In Testing", "QE Review")
     AND labels NOT IN ("aicp-team-forge", "aicp-team-compass", "aicp-team-heimdall", "no-subteam", "needs-subteam")
Fields: key, summary, status, assignee, labels
```

## Output Format

### 7-Section Document

#### Section 1: What We Own

Two parts: **Owned roadmap features** (from spreadsheet Roadmap tab, Ownership = "AICP") and **engineering portfolio** (from Jira queries).

**Owned Roadmap Features** — table from spreadsheet:

| Feature | Title | Status | Release | Sub-Team | Priority | Effort |
|---------|-------|--------|---------|----------|----------|--------|
| {key} | {title} | {status} | {release} | {team} | {priority} | {effort} |

**Platform Support** — features where Ownership = "Support" (AICP reviews/consults but doesn't own):

| Feature | Title | Status | AICP Work | Sub-Team |
|---------|-------|--------|-----------|----------|
| {key} | {title} | {status} | {supporting_work} | {team} |

**Engineering Portfolio** — sub-team table from Jira data:

| Sub-Team | Focus Area | In-Flight | Resolved ({N}d) |
|----------|-----------|-----------|-----------------|
| Forge | xKS Expansion, GitOps & CI/CD | {count} | {count} |
| Compass | E2E Stability, QE & Observability | {count} | {count} |
| Heimdall | Security, Gateway & Authentication | {count} | {count} |
| Unassigned | No sub-team label | {count} | {count} |
| **Total** | | **{sum}** | **{sum}** |

Include total portfolio size from Query 1. Frame as: "AICP owns {N} roadmap features and supports {M} cross-team initiatives, with {P} open engineering issues across {Q} sub-teams."

#### Section 2: What We've Delivered

Resolved issues grouped by theme (from Query 3). For each theme:
- Theme name as subheading
- Count of resolved issues
- 3-5 key items with outcome-focused descriptions
- Jira links: `https://redhat.atlassian.net/browse/{KEY}`

For component onboarding (CLONE) tickets, group separately:
"Completed {N} component onboarding tasks for {Component A}, {Component B}, etc."

#### Section 3: What's In Flight

Active issues by sub-team (from Query 2). For each team:
- Team name and focus area as subheading
- Count of active issues
- Top 3-5 key items with brief descriptions
- Include priority and assignee where relevant

#### Section 4: Customer Signal

From Queries 4 and 5:
- RHOAIENG customer issues (if any) — likely zero for AICP component
- RHAISTRAT field requests and customer-driven features
- Be honest: "AICP component has {N} direct customer-labeled issues. Customer impact is typically indirect — AICP platform stability underpins all RHOAI components."

#### Section 5: Where We See Opportunity

Derived analysis — not a direct Jira query. Synthesize from:
- Unassigned work (Query 7) — areas without ownership
- Themes with high resolved counts but no strategic alignment
- RHAISTRAT features that could benefit from AICP investment
- Gaps between team capacity and incoming work

Frame as actionable opportunities, not complaints.

#### Section 6: What We Need

Capacity, ownership, and process asks for leadership. Derive from:
- Unassigned issue count and themes
- Blocker/critical issues needing escalation (Query 6)
- Resource constraints visible in the data
- Include user-provided context from Prompt 3

#### Section 7: Next Steps

Action/timeline/owner table:

| Action | Owner | Timeline |
|--------|-------|----------|
| {action item} | {name or team} | {date or "Next sprint"} |

Derive from blockers needing resolution, unassigned work needing triage, and upcoming milestones visible in the data.

## Word Document Generation

Use `python-docx` to generate the Word document:

```python
from docx import Document
from docx.shared import Inches, Pt, Cm, RGBColor
from docx.oxml.ns import qn, nsdecls
from docx.oxml import parse_xml
```

### Styling

- **Title**: 16pt, bold, dark gray (#333333)
- **Section headers**: 14pt, bold, Red Hat red (#CC0000)
- **Body text**: 11pt, Calibri
- **Table headers**: Red Hat red (#CC0000) background, white text, bold
- **Table alternating rows**: Light gray (#F2F2F2) on even rows
- **Table borders**: Thin black borders on all cells

### Table Border Helper

```python
def set_table_borders(table):
    tbl = table._tbl
    tblPr = tbl.tblPr if tbl.tblPr is not None else parse_xml(f'<w:tblPr {nsdecls("w")}/>')
    borders = parse_xml(
        f'<w:tblBorders {nsdecls("w")}>'
        '  <w:top w:val="single" w:sz="4" w:space="0" w:color="000000"/>'
        '  <w:left w:val="single" w:sz="4" w:space="0" w:color="000000"/>'
        '  <w:bottom w:val="single" w:sz="4" w:space="0" w:color="000000"/>'
        '  <w:right w:val="single" w:sz="4" w:space="0" w:color="000000"/>'
        '  <w:insideH w:val="single" w:sz="4" w:space="0" w:color="000000"/>'
        '  <w:insideV w:val="single" w:sz="4" w:space="0" w:color="000000"/>'
        '</w:tblBorders>'
    )
    tblPr.append(borders)
```

### Row Shading Helper

```python
def shade_cells(row, color_hex):
    for cell in row.cells:
        shading = parse_xml(f'<w:shd {nsdecls("w")} w:fill="{color_hex}"/>')
        cell._tc.get_or_add_tcPr().append(shading)
```

## Output Locations

- Word: `docs/strategy/{YYYY-MM-DD}_strategy_review.docx`
- Markdown: `docs/strategy/{YYYY-MM-DD}_strategy_review.md`

Where `{YYYY-MM-DD}` is today's date.

After generation, display a summary of findings to the user and ask:
1. "Would you like to revise anything?"
2. "Would you like to commit these files?"

## Quality Checks

Before presenting the report, verify:

- [ ] Roadmap features correctly split by Ownership column: "AICP" = owned, "Support" = supporting
- [ ] All Jira data scoped to `component in ("AI Core Platform", "AI Core Platform Security")` only — no Model Serving, Dashboard, MLflow, AI Pipelines, or other components
- [ ] Sub-teams derived from `aicp-team-*` labels only — never infer team from assignee or other fields
- [ ] Customer signal section is honest about AICP-direct vs downstream (zero direct is normal)
- [ ] In-flight total = sum of all sub-team counts + unassigned count
- [ ] Resolved total = sum of all sub-team resolved counts + unassigned resolved count
- [ ] All Jira links use `https://redhat.atlassian.net/browse/{KEY}` format
- [ ] RHAISTRAT items are clearly labeled as RHAISTRAT scope, not mixed with RHOAIENG
- [ ] Word doc has Red Hat red (#CC0000) table headers with white text
- [ ] No downstream component issues leaked in — spot-check 3-5 issues against Jira

## Troubleshooting

### Authentication fails
- Read `/Users/jaykoehler/.claude.json` for the current token
- Token is in `mcpServers.mcp-atlassian.args` array, value after `--jira-token`
- Email is the value after `--jira-username`
- If token is expired, ask the user to update their Jira PAT

### Empty results from queries
- Verify the component filter is exactly `"AI Core Platform"` (case-sensitive)
- Check that the project key is `RHOAIENG` not `RHOAISTRAT` (that project doesn't exist)
- For RHAISTRAT queries, note that component filter is NOT applied

### total: -1 in API response
- Normal for complex JQL (especially `status changed to ... DURING` clauses)
- Do NOT rely on the `total` field — paginate and count results manually

### python-docx not available
- Install: `pip install python-docx`
- The `lxml` dependency is required for table border/shading XML manipulation

### No Crucible sub-team label
- Despite documentation listing 4 sub-teams, `aicp-team-crucible` does not exist in Jira data
- Only Forge, Compass, and Heimdall labels are in active use
