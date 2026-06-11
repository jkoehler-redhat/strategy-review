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
  2. PowerPoint deck (.pptx)
  3. Both Word + PowerPoint
  4. Markdown (.md)
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

### 8-Section Document

#### Executive Summary (Section 0)

A bullet-point overview at the top of the document with 5-8 data-backed bullets. Each bullet references a detail section via an in-document bookmark link (clickable in Word). The summary gives leadership the full picture in 30 seconds — the rest of the document is supporting detail.

**Bullet structure** — derive from collected data, covering these areas:

1. **Portfolio scope** — "{N} roadmap features owned, {M} cross-team support items, {P} open engineering issues across 3 sub-teams" → links to Section 1
2. **Delivery velocity** — "{X} issues resolved in {N} days: {A} strategic initiatives, {B} bugs/security fixes, {C} feature stories, {D} operational tasks" → links to Section 2
3. **Top delivery highlight** — One sentence on the most impactful completed initiative or theme → links to Section 2
4. **In-flight snapshot** — "{Y} issues actively in progress across Forge ({f}), Compass ({c}), Heimdall ({h})" → links to Section 3
5. **Blockers & risks** — "{Z} blocker/critical issues open" with the top 1-2 named → links to Section 6
6. **Customer signal** — Brief framing of direct vs indirect customer impact → links to Section 4
7. **Key opportunity** — One sentence on the biggest opportunity identified → links to Section 5
8. **Top ask** — The single most important leadership ask → links to Section 6

Not every bullet is required — include only those backed by meaningful data. Minimum 5 bullets.

**Word document implementation:**

Each detail section heading gets a bookmark (e.g., `section_1_what_we_own`, `section_2_delivered`). Each executive summary bullet includes a hyperlink to the relevant bookmark so readers can click to jump to details.

```python
from docx.oxml.ns import qn, nsdecls
from docx.oxml import parse_xml, OxmlElement

def add_bookmark(paragraph, bookmark_name):
    """Add a bookmark anchor to a paragraph (the target)."""
    tag = paragraph._p
    bookmark_start = OxmlElement('w:bookmarkStart')
    bookmark_start.set(qn('w:id'), str(hash(bookmark_name) % 10000))
    bookmark_start.set(qn('w:name'), bookmark_name)
    bookmark_end = OxmlElement('w:bookmarkEnd')
    bookmark_end.set(qn('w:id'), str(hash(bookmark_name) % 10000))
    tag.insert(0, bookmark_start)
    tag.append(bookmark_end)

def add_internal_hyperlink(paragraph, bookmark_name, link_text):
    """Add a clickable hyperlink within the document to a bookmark."""
    hyperlink = OxmlElement('w:hyperlink')
    hyperlink.set(qn('w:anchor'), bookmark_name)
    run = OxmlElement('w:r')
    rPr = OxmlElement('w:rPr')
    color = OxmlElement('w:color')
    color.set(qn('w:val'), '0563C1')
    underline = OxmlElement('w:u')
    underline.set(qn('w:val'), 'single')
    rPr.append(color)
    rPr.append(underline)
    run.append(rPr)
    text = OxmlElement('w:t')
    text.text = link_text
    run.append(text)
    hyperlink.append(run)
    paragraph._p.append(hyperlink)
```

**Bookmark names:**

| Section | Bookmark |
|---------|----------|
| 1. What We Own | `section_1_what_we_own` |
| 2. What We've Delivered | `section_2_delivered` |
| 3. What's In Flight | `section_3_in_flight` |
| 4. Customer Signal | `section_4_customer_signal` |
| 5. Where We See Opportunity | `section_5_opportunity` |
| 6. What We Need | `section_6_what_we_need` |
| 7. Next Steps | `section_7_next_steps` |

Each section heading paragraph must call `add_bookmark()` with its bookmark name. Each executive summary bullet uses `add_internal_hyperlink()` to append a " → See details" link pointing to the relevant bookmark.

### Document Structure Principle

**Narrative body, reference appendix.** Sections 1-7 are written as a briefing — concise narratives, outcome-focused bullet points, and summary counts. No raw Jira issue tables in the body. All issue-level detail (keys, titles, assignees) goes in a Reference Appendix at the end, organized by category. Each body section ends with a note pointing to the appendix.

Page breaks between: Executive Summary → Sections 1-3 → Section 5 → Appendix.

#### Section 1: What We Own

Narrative intro: "AICP owns {N} roadmap features and provides platform support for {M} cross-team initiatives. The engineering portfolio comprises {P} open issues distributed across three sub-teams."

Three parts with **curated tables** (these are roadmap/summary tables, not raw Jira dumps):

1. **Owned Roadmap Features** — table from spreadsheet (Ownership = "AICP")
2. **Platform Support Engagements** — table from spreadsheet (Ownership = "Support")
3. **Engineering Portfolio by Sub-Team** — aggregated counts table (Forge/Compass/Heimdall/Unassigned with in-flight and resolved counts)

#### Section 2: What We've Delivered

Narrative summary of outcomes, NOT a list of tickets. For each issue type category:

| Jira Issue Type | Report Category |
|----------------|-----------------|
| Epic, Initiative | Strategic Initiatives |
| Bug, Vulnerability, Weakness | Bug Fixes & Security |
| Story, Task | Feature & Engineering Work |
| Sub-task + "CLONE -" summaries | Component Onboarding & Operational |

For each category:
- **Strategic**: Count + bullet points grouped by theme with outcome descriptions (not ticket titles)
- **Bug Fixes**: Count + theme breakdown with CVE count callout per theme
- **Feature Work**: Count + top themes with representative outcome per theme
- **Operational**: Count + component onboarding summary

End with: "Full issue-level detail is available in the Reference Appendix."

#### Section 3: What's In Flight

Narrative by sub-team (NOT by issue type — team is the primary lens for in-flight). For each sub-team:
- Team name, focus area, and count as subheading
- Top 3-4 items shown as outcome-focused bullets (prioritize strategic initiatives and blocker/critical items)
- No raw Jira tables

End with: "Complete in-flight issue listings are in the Reference Appendix."

#### Section 4: Customer Signal

Narrative framing of direct vs indirect customer impact:
- State AICP direct customer count (typically zero) with honest explanation
- RHAISTRAT customer items: total count + how many touch AICP platform capabilities
- Cross-reference RHAISTRAT items against AICP platform themes (auth, multi-cloud, operator, security, observability, batch/kueue, gitops) — list the AICP-relevant ones by theme
- No raw issue table — reference appendix has the full list

#### Section 5: Where We See Opportunity

**Data-driven strategic analysis** — not a restatement of other sections. Derive from cross-referencing:

1. **Customer demand → AICP investment**: Cross-reference RHAISTRAT customer-labeled items against AICP platform themes. Cluster by theme and surface where customer demand is highest. Frame as: investment here unblocks multiple downstream teams.

2. **Owned features without active work**: Compare owned roadmap features (spreadsheet) against in-flight issues (Query 2). Features with status "In Progress" or "New" but no matching Jira engineering work are stalled — flag for resourcing or reprioritization.

3. **Recurring bug themes suggesting underinvestment**: Themes with disproportionately high bug volume suggest areas where proactive engineering (test coverage, architecture) could reduce maintenance burden.

4. **Ownership gaps**: In-flight issues without sub-team labels (Query 7) represent work happening outside the team structure.

Frame each opportunity as: what it is, why it matters, what AICP would need to do.

#### Section 6: What We Need

**Asks tied to opportunities.** Each need connects back to data from other sections:

1. **Blocker resolution support** — count + framing of delivery risk
2. **Roadmap prioritization review** — stalled features need resourcing, descoping, or dependency unblocking
3. **Strategic alignment on customer-driven work** — which RHAISTRAT customer items should become AICP roadmap commitments
4. **Triage support** — unassigned issues need sub-team assignment

Include user-provided context from Prompt 3.

#### Section 7: Next Steps

Action/timeline/owner table derived from the needs above. Each action is specific and tied to a data point.

### Reference Appendix

All raw Jira issue tables go here, organized as:

| Section | Content |
|---------|---------|
| A1 | Strategic Initiatives Delivered |
| A2 | Bug Fixes & Security Resolved |
| A3 | Feature & Engineering Work Resolved |
| A4 | Operational Tasks Resolved |
| B1 | Strategic Initiatives In Flight |
| B2 | Active Bugs |
| B3 | Feature Work In Flight |
| B4 | Operational In Flight |
| C | Blocker & Critical Issues |
| D | RHAISTRAT Customer & Field Items (with AICP-relevance flag) |

Each appendix table uses full Jira keys, full titles (no truncation), and standard Red Hat red headers.

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

## PowerPoint Deck Generation

When the user selects PowerPoint output, generate a 14-slide widescreen leadership deck using `python-pptx`. The reference format is the Model Serving & Safety team's strategy review deck presented to Sherard Griffin.

### python-pptx Setup

On macOS managed Python environments, `python-pptx` may not be on the default path. Add the homebrew site-packages:

```python
import sys
sys.path.insert(0, '/opt/homebrew/lib/python3.13/site-packages')
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN, MSO_ANCHOR
from pptx.enum.shapes import MSO_SHAPE
```

If import fails, install: `pip3 install --break-system-packages python-pptx`

### Slide Layout

Widescreen 16:9 (13.333 x 7.5 inches). Content slides use blank layout with a thin red (#CC0000) bar at the top (0.07" tall). Section divider slides are full-bleed dark gray with a red left accent bar.

| Color | Hex | Usage |
|-------|-----|-------|
| Red Hat Red | #CC0000 | Top bar, headers, table headers, accent bar, bet column labels |
| Dark Gray | #333333 | Title text, body text, section bg |
| Medium Gray | #666666 | Subtitles, labels, breadcrumb text |
| White | #FFFFFF | Text on red/dark backgrounds |
| Light Gray | #F2F2F2 | Alternating table rows |

### 14-Slide Structure

Three sections separated by full-bleed section divider slides:

**Section 1: Portfolio Map** — What AICP owns, what we support, and where we're investing in platform quality

| Slide | Title | Content |
|-------|-------|---------|
| 1 | Title | "AI Core Platform" / "Strategy Review" / date / presenter |
| 2 | Agenda | 3-item agenda matching section structure, ~10 min format note |
| 3 | [Section: Portfolio Map] | Dark gray full-bleed divider |
| 4 | Owned Roadmap Features | 3-column layout: Forge / Compass / Heimdall with status dots and release pills |
| 5 | Platform Support Engagements | Headline stat + sub-team summary table + nearest-term callouts (see detail below) |

**Section 2: Traction** — What's shipped, what's working, and where we're investing

| Slide | Title | Content |
|-------|-------|---------|
| 6 | [Section: Traction] | Dark gray full-bleed divider |
| 7 | Platform Upgrade Investment | RHAISTRAT-1519 narrative + supporting RHOAIENG issues table |
| 8 | What We've Shipped | Resolved RHAISTRAT count + themes with keys |

**Section 3: Strategic Opportunities** — Three bets backed by customer data

| Slide | Title | Content |
|-------|-------|---------|
| 9 | [Section: Strategic Opportunities] | Dark gray full-bleed divider |
| 10 | Bet 1: Multi-Tenancy | Two-column bet slide (see detail below) |
| 11 | Bet 2: GPUaaS / DRA | Two-column bet slide |
| 12 | Bet 3: Automated Upgrade Validation | Two-column bet slide |
| 13 | What We Need | 5 data-backed asks tied to evidence from earlier slides |
| 14 | Thank You | Open discussion close |

### Section Divider Slides

Full-bleed dark gray (#333333) background with a red (#CC0000) left accent bar (0.15" wide), large white title centered vertically, optional gray subtitle.

```python
def section_slide(title, subtitle=None):
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    bg = slide.shapes.add_shape(MSO_SHAPE.RECTANGLE, Inches(0), Inches(0), prs.slide_width, prs.slide_height)
    bg.fill.solid(); bg.fill.fore_color.rgb = DKGRAY; bg.line.fill.background()
    accent = slide.shapes.add_shape(MSO_SHAPE.RECTANGLE, Inches(0), Inches(0), Inches(0.15), prs.slide_height)
    accent.fill.solid(); accent.fill.fore_color.rgb = RED; accent.line.fill.background()
    # 40pt white bold title at y=2.8, 18pt gray subtitle at y=4.0
```

### Slide 4: Owned Roadmap Features (3-Column)

Three columns: Forge, Compass, Heimdall. Each column has a red header box with team name and count. Items listed with:
- Status dot: green = In Progress, orange = New/Backlog, blue = Review/Release Pending
- Feature title (full length, word wrap enabled)
- Release version pill right-aligned (e.g. "3.5")

Source: spreadsheet `Ownership = AICP`, grouped by `sub_team` column (J).

### Slide 5: Platform Support Engagements

**Do not list all support items.** The slide has three parts:

1. **Headline stat**: `"{N} support engagements  ·  {M} owned features"` at 18pt bold, followed by italic framing: "Nearly half the AICP portfolio is cross-team support — a capacity and strategic focus tradeoff worth aligning on."

2. **Sub-team summary table** (4 rows max): Sub-Team / # / Releases / Representative Example. Group support items by sub_team, show count, list target releases, pick one representative title.

3. **Nearest-term callouts**: 3 bullet items for the support items with the closest release target. Format: `{KEY}  {title}  ·  {sub_team}  ·  {release}`

### Bet Slides (Slides 10-12)

Two-column layout with a horizontal divider below the title:

- **Left column**: "The market signal" (red bold label) — named customers, deal counts, geographic segments, competitive risk. Sourced entirely from RHAISTRAT ADF description text.
- **Right column**: "Why this is ours to win" (red bold label) — what AICP owns, why it's a platform-layer problem, current state and gap.
- Vertical gray divider between columns at x=6.55"

```python
def bet_slide(title, signal_text, win_text, subtitle=None):
    slide = blank_slide(title, subtitle)
    # horizontal divider at y=1.25
    # left: "The market signal" label at (0.6, 1.35), text at (0.6, 1.75)
    # vertical divider at (6.55, 1.35)
    # right: "Why this is ours to win" label at (6.7, 1.35), text at (6.7, 1.75)
```

Bet content is derived from real RHAISTRAT descriptions using `extract_text()` on ADF JSON. Never invent customers, deal counts, or customer names.

Current bets (update from live data each run):
- **Bet 1**: Multi-Tenancy (RHAISTRAT-1471) — 20+ Nordic CCSPs, BBVA/Telenor/Aramco/SWIFT
- **Bet 2**: GPUaaS / DRA (RHAISTRAT-1470) — 5+ EMEA opportunities on OCP 4.21
- **Bet 3**: Automated Upgrade Validation (RHAISTRAT-1519) — release quality gate

### RHAISTRAT Description Mining

Fetch description for each key RHAISTRAT item using REST API (`fields=summary,status,priority,description`). Parse the ADF JSON with:

```python
def extract_text(node):
    if not node: return ''
    t = node.get('type', ''); content = node.get('content', [])
    if t == 'text':
        text = node.get('text', '')
        for mark in node.get('marks', []):
            if mark.get('type') == 'link':
                href = mark.get('attrs', {}).get('href', '')
                if href and href not in text:
                    text = f"{text} ({href})"
        return text
    if t == 'inlineCard':
        return node.get('attrs', {}).get('url', '')
    if t in ('paragraph', 'heading'): return ' '.join(extract_text(c) for c in content).strip() + '\n'
    if t in ('bulletList', 'orderedList'): return ''.join(extract_text(c) for c in content)
    if t == 'listItem': return '• ' + ' '.join(extract_text(c) for c in content).strip() + '\n'
    if t == 'hardBreak': return '\n'
    return ''.join(extract_text(c) for c in content)
```

**Critical**: Without handling `marks` for link nodes and `inlineCard` type, URLs disappear from descriptions — text like "For design, see the RFC:" renders with nothing after the colon.

Look for in the extracted text: named customers, deal counts, geographic segments, competitive risks, and what is blocked without the capability. Use this verbatim (cleaned) as bet slide content — do not paraphrase or invent.

### Slide 13: What We Need

Five data-backed asks, each tied to evidence from earlier slides:
1. Multi-Tenancy resourcing (RHAISTRAT-1471 Critical, 0 active engineering)
2. DRA / GPUaaS engineering alignment (RHAISTRAT-1470 In Progress, only backlog issues)
3. Upgrade CI infrastructure (RHAISTRAT-1519 + supporting RHOAIENG count)
4. Blocker escalation (blocker count + critical count + top named blockers)
5. Sub-team triage — in-flight issues without any `aicp-team-*` label need ownership assignment

Needs come **only from data** — never invented. Blocker names cleaned: strip `[tag]` prefixes, "- N week notice!" suffixes, truncate to 60 chars.

### Helper Functions

```python
RED   = RGBColor(0xCC, 0x00, 0x00)
DKGRAY= RGBColor(0x33, 0x33, 0x33)
MDGRAY= RGBColor(0x66, 0x66, 0x66)
WHITE = RGBColor(0xFF, 0xFF, 0xFF)
LTGRAY= RGBColor(0xF2, 0xF2, 0xF2)

def blank_slide(title_text=None, subtitle=None):
    """Blank slide with red top bar, optional title (26pt bold) and subtitle (12pt italic gray)."""

def add_text_box(slide, text, left, top, width, height, size=13, bold=False, color=None, italic=False, wrap=True):
    """Single-run text box."""

def add_bullet_box(slide, items, left, top, width, height, size=13, space_after=5):
    """Multi-paragraph text box. items = [(bold_prefix, rest), ...]"""

def add_table(slide, headers, rows, left, top, width, row_h=0.38, font_size=11):
    """Table with red headers, white header text, alternating light gray rows."""

def label_pill(slide, text, left, top, color=RED, text_color=WHITE, width=1.1, height=0.28, size=10):
    """Rounded rectangle label (GA/TP/DP maturity badges)."""
```

Prioritize items: strategic initiatives first, then blocker/critical priority, then remaining. Show full item summary with priority tag (no truncation — word wrap handles overflow).

## Output Locations

- Word: `docs/strategy/{YYYY-MM-DD}_strategy_review.docx`
- PowerPoint: `docs/strategy/{YYYY-MM-DD}_strategy_review.pptx`
- Markdown: `docs/strategy/{YYYY-MM-DD}_strategy_review.md`
- Google Doc: `https://docs.google.com/document/d/1kU_huAxuzmx3wMm34n2FsleMLWs_0TC5/edit`

Where `{YYYY-MM-DD}` is today's date.

### Google Doc Upload

After generating the local file, push it to the shared Google Doc:

```bash
gws drive files update \
  --params '{"fileId":"1kU_huAxuzmx3wMm34n2FsleMLWs_0TC5"}' \
  --upload "docs/strategy/{YYYY-MM-DD}_strategy_review.docx" \
  --upload-content-type "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
```

**Important**: The `--upload` path must be relative to the current working directory (gws rejects absolute paths outside cwd). Copy the file into cwd if needed before uploading.

**Scope requirement**: The `gws` tool must have `https://www.googleapis.com/auth/drive` scope (not just `drive.readonly`). If upload fails with `insufficientPermissions`, re-auth with write scopes:

```bash
gws auth login --scopes "https://www.googleapis.com/auth/drive,https://www.googleapis.com/auth/documents,https://www.googleapis.com/auth/calendar.readonly"
```

If it still fails after re-auth, delete the stale token cache and retry:

```bash
rm -f ~/.config/gws/token_cache.json
```

After generation, display a summary of findings to the user and ask:
1. "Would you like to revise anything?"
2. "Would you like to commit these files?"

## AI Hub Deck (`ai_hub_pptx.py`)

A separate 13-slide PowerPoint for the AI Hub component — same structure as the AICP deck but scoped differently.

**Component scope**: `component = "AI Hub"` in RHOAIENG (not AI Core Platform)

**No sub-team labels**: AI Hub uses theme-based grouping instead of `aicp-team-*` labels.

**No release gate slides**: AI Hub doesn't have an upgrade/release gate narrative.

### Theme Mapping

| Theme | Keywords |
|-------|---------|
| Model Catalog | catalog, eagle3, xeon, tool calling, text-embedding, cold-start, vram, benchmark, model batch |
| MCP Registry | mcp registry, mcp server |
| MLflow / Asset Registry | mlflow, unity catalog, asset registry, tiger team, ai asset |
| Bodies of Water | bodies of water, lake, ocean, stream, modular upgrade |
| Security & CVEs | cve, vulnerability, schemathesis, ssrf, hermetic, fuzz, signing, cosign, securesign |
| Async Upload / OCI | async upload, async-upload, omlmd, oci, image sign |
| Upstream / Community | kubeflow, upstream, mlmd, graduation, community, blog |

### 13-Slide Structure

| Slide | Title | Notes |
|-------|-------|-------|
| 1 | Title | "AI Hub" / "Strategy Review" |
| 2 | Agenda | 3 sections |
| 3 | [Section: Portfolio Map] | Divider |
| 4 | Portfolio Overview | Theme breakdown table (open, in-flight, resolved per theme) |
| 5 | In Flight | Themes × top 3 summaries per theme (no truncation) |
| 6 | [Section: Traction] | Divider |
| 7 | What We've Shipped | Resolved by theme with issue keys |
| 8 | [Section: Strategic Opportunities] | Divider |
| 9 | Bet 1: MLflow / Asset Registry | RHOAIENG-50747 |
| 10 | Bet 2: MCP Registry | RHOAIENG-63382 / RHOAIENG-63350 |
| 11 | Bet 3: Decouple Model Catalog | RHOAIENG-60367 |
| 12 | What We Need | 4 asks: MLflow decision, MCP resourcing, CVE backlog, priority triage |
| 13 | Thank You | Open discussion |

### What We Need (AI Hub — 4 bullets)

1. MLflow architecture decision (tiger team research complete, RFC in progress)
2. MCP Registry resourcing to hit Tech Preview in 3.5
3. CVE backlog prioritization (10 open CVEs)
4. Priority triage — ~55% of portfolio is Undefined priority

### Output

- `docs/strategy/{YYYY-MM-DD}_ai_hub_strategy_review.pptx`
- `~/ai-hub-strategy-review.pptx` (convenience copy)

Run: `PYTHONPATH=/opt/homebrew/lib/python3.13/site-packages python3 ai_hub_pptx.py`

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

### python-pptx not available or import fails
- On macOS managed Python, add to script: `sys.path.insert(0, '/opt/homebrew/lib/python3.13/site-packages')`
- Install: `pip3 install --break-system-packages python-pptx`
- PEP 668 blocks `pip install --user` on managed environments — use `--break-system-packages`

### No Crucible sub-team label
- Despite documentation listing 4 sub-teams, `aicp-team-crucible` does not exist in Jira data
- Only Forge, Compass, and Heimdall labels are in active use
