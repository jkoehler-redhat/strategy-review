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

When the user selects PowerPoint output, generate an 8-slide widescreen leadership deck using `python-pptx`.

### python-pptx Setup

On macOS managed Python environments, `python-pptx` may not be on the default path. Add the homebrew site-packages:

```python
import sys
sys.path.insert(0, '/opt/homebrew/lib/python3.13/site-packages')
from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN, MSO_ANCHOR
from pptx.enum.shapes import MSO_SHAPE
```

If import fails, install: `pip3 install --break-system-packages python-pptx`

### Slide Layout

Widescreen 16:9 (13.333 x 7.5 inches). All slides use blank layout with a thin red (#CC0000) bar at the top (0.08" tall). Red Hat branding colors:

| Color | Hex | Usage |
|-------|-----|-------|
| Red Hat Red | #CC0000 | Top bar, KPI numbers, table headers, accent elements |
| Dark Gray | #333333 | Title text, body text |
| Medium Gray | #666666 | Subtitles, labels |
| White | #FFFFFF | Text on red backgrounds |
| Light Gray BG | #F2F2F2 | KPI boxes, alternating table rows |

### 8-Slide Structure

| Slide | Title | Content |
|-------|-------|---------|
| 1 | Title | "AI Core Platform" / "Strategy Review" / date + lookback period / "Red Hat OpenShift AI" |
| 2 | Executive Summary | Full-sentence bullets with bold labels — Portfolio, Delivered, In Flight, Blockers, Customer Signal, Gap (see detail below) |
| 3 | What We Own | 4 KPI boxes (Owned Features, Support Engagements, Open Issues, Sub-Teams) + sub-team summary table |
| 4 | What We've Delivered | 4 KPI boxes (Strategic Epics, Bug/Security, Feature, Operational) + top 5 themes by volume with bug counts |
| 5 | What's In Flight | 3-column layout — Forge/Compass/Heimdall with red header boxes, focus area subtitle, top 4 items tagged ★ strategic / ! blocker-critical / · other |
| 6 | Customer Signal | Bullet list: direct AICP count + RHAISTRAT total + AICP-relevant count + top 5 AICP-relevant items with keys and theme tags |
| 7 | Strategic Opportunities | 3-column table: Opportunity / Current State (from Jira) / Why It Matters (from RHAISTRAT descriptions) |
| 8 | What We Need & Next Steps | Data-backed needs bullets + placeholder action table for owner/timeline (filled in before meeting) |

### Slide 2: Executive Summary Detail

Each bullet is a **full sentence**, not a data dump. Bold label followed by narrative. Use this structure:

```
Portfolio  — "{N} owned roadmap features and {M} cross-team support engagements, backed by {P} open engineering issues across Forge ({f}), Compass ({c}), and Heimdall ({h}) in-flight today."

Delivered ({N}d)  — "{X} issues closed — {A} strategic epics, {B} bugs and CVEs, {C} feature stories. Highest volume: {top 3 themes with counts}."

In Flight  — "{Y} issues actively in progress — {A} strategic initiatives, {B} features, {C} bugs."

Blockers  — "{N} blocker-priority and {M} critical issues remain open. Top blockers: {clean name 1}; {clean name 2}."

Customer Signal  — "Platform impact is primarily indirect — {N} of {M} open RHAISTRAT field requests map directly to AICP platform capabilities."

Gaps  — "{N} owned roadmap features have no active engineering work, including DRA and multi-tenancy — both critical for {release} delivery." (only if stalled_features > 0)
```

**Blocker name cleaning**: Strip `[tag]` prefixes, "- N week notice!" suffixes, replace " — " with ": ". Truncate to 60 chars.

### Slide 7: Strategic Opportunities — RHAISTRAT Description Mining

The "Why It Matters" column is derived from the **actual RHAISTRAT issue descriptions**, not invented. For each owned roadmap feature, fetch the full description via REST API and extract:
- **Affected Customers** section — named accounts, segments, deal counts
- **Problem Statement** — what breaks without AICP investment
- **Business Alignment** — named deals, POCs, urgency signals

```python
def extract_text(node):
    """Recursively extract plain text from Jira ADF description node."""
    if not node: return ''
    if isinstance(node, str): return node
    t = node.get('type', '')
    content = node.get('content', [])
    if t == 'text': return node.get('text', '')
    if t in ('paragraph', 'heading'): return ' '.join(extract_text(c) for c in content).strip() + '\n'
    if t in ('bulletList', 'orderedList'): return ''.join(extract_text(c) for c in content)
    if t == 'listItem': return '• ' + ' '.join(extract_text(c) for c in content).strip() + '\n'
    if t == 'hardBreak': return '\n'
    return ''.join(extract_text(c) for c in content)

def get_strat_description(key):
    url = f"{BASE_URL}/rest/api/3/issue/{key}?fields=summary,status,priority,description"
    req = urllib.request.Request(url, headers={"Authorization": f"Basic {AUTH}", "Content-Type": "application/json"})
    with urllib.request.urlopen(req) as r:
        data = json.loads(r.read())
    return extract_text(data["fields"].get("description") or {})
```

Fetch descriptions for each owned roadmap feature's RHAISTRAT key. Parse the text to pull the most useful 1–2 sentences for the "Why It Matters" column. Look for: named customers, deal counts, geographic segments, competitive risk, and what is blocked without the capability.

**Example framing (derived from real description text):**

| Opportunity | Current State | Why It Matters |
|-------------|---------------|----------------|
| GPUaaS / DRA | RHAISTRAT-1470 \| In Progress \| 0 active / 2 backlog | 5+ EMEA opportunities on OCP 4.21 blocked by whole-GPU-only allocation. RHOAI components (KServe, notebooks, KubeRay) must emit DRA ResourceClaims to unlock GPU fractions, sharing, and multi-device topology. AICP owns this — engineering investment is critical path. |
| Multi-tenancy | RHAISTRAT-1471 \| Critical \| 0 active engineering | 20+ Nordic CCSPs (Atea, Advania, Volvo, AI Sweden) cannot commercialize RHOAI AI Factory without tenant isolation. BBVA, Telenor, Aramco, SWIFT need chargeback and namespace governance. Competitive gap vs VMware. No engineering work started. |
| Kueue / Batch | RHAISTRAT-1477 \| 4 active / 13 backlog | Truist (active Spark PoC) and BNY named Kueue/Spark as a blocker for at-scale adoption. GPU contention and underutilization block financial services customers without it. |
| xKS / Multi-Cloud | RHAISTRAT-1209 \| 0 active / 12 backlog | 2 customers (AKS, Coreweave) waiting on GA of llm-d distributed inference on non-OCP Kubernetes. Auth/gateway for xKS unresolved — blocking 3.5 GA. |

### Slide 8: What We Need & Next Steps

Needs bullets come **only from data**:
- Blocker count (if > 0)
- Stalled owned features count (if > 0)
- Unassigned in-flight count (if > 0)

Next Steps table uses **placeholder rows** — the manager fills in owner and timeline before the meeting. Do not invent owners or timelines.

### Helper Functions

```python
RED = RGBColor(0xCC, 0x00, 0x00)
DARK_GRAY = RGBColor(0x33, 0x33, 0x33)
MED_GRAY = RGBColor(0x66, 0x66, 0x66)
WHITE = RGBColor(0xFF, 0xFF, 0xFF)
LIGHT_GRAY_BG = RGBColor(0xF2, 0xF2, 0xF2)

def add_slide(prs, title_text=None):
    """Blank slide with red top bar and optional title."""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    shape = slide.shapes.add_shape(MSO_SHAPE.RECTANGLE, Inches(0), Inches(0), prs.slide_width, Inches(0.08))
    shape.fill.solid(); shape.fill.fore_color.rgb = RED; shape.line.fill.background()
    if title_text:
        txBox = slide.shapes.add_textbox(Inches(0.6), Inches(0.3), Inches(12), Inches(0.7))
        tf = txBox.text_frame; tf.word_wrap = True
        p = tf.paragraphs[0]
        run = p.add_run(); run.text = title_text
        run.font.size = Pt(28); run.font.bold = True; run.font.color.rgb = DARK_GRAY; run.font.name = "Calibri"
    return slide

def add_kpi_boxes(slide, kpis, top=1.3):
    """KPI number boxes. kpis = [(number, label), ...]"""
    n = len(kpis); box_width = 2.5; gap = 0.3
    total_width = n * box_width + (n-1) * gap
    start_x = (13.333 - total_width) / 2
    for i, (number, label) in enumerate(kpis):
        x = start_x + i * (box_width + gap)
        shape = slide.shapes.add_shape(MSO_SHAPE.ROUNDED_RECTANGLE, Inches(x), Inches(top), Inches(box_width), Inches(1.5))
        shape.fill.solid(); shape.fill.fore_color.rgb = LIGHT_GRAY_BG
        shape.line.color.rgb = RGBColor(0xDD, 0xDD, 0xDD); shape.line.width = Pt(1)
        txBox = slide.shapes.add_textbox(Inches(x), Inches(top + 0.15), Inches(box_width), Inches(0.8))
        p = txBox.text_frame.paragraphs[0]; p.alignment = PP_ALIGN.CENTER
        run = p.add_run(); run.text = str(number)
        run.font.size = Pt(36); run.font.bold = True; run.font.color.rgb = RED; run.font.name = "Calibri"
        txBox2 = slide.shapes.add_textbox(Inches(x), Inches(top + 0.85), Inches(box_width), Inches(0.6))
        p2 = txBox2.text_frame.paragraphs[0]; p2.alignment = PP_ALIGN.CENTER
        run2 = p2.add_run(); run2.text = label
        run2.font.size = Pt(13); run2.font.color.rgb = MED_GRAY; run2.font.name = "Calibri"

def add_bullet_list(slide, items, left=0.6, top=1.2, width=12, height=5.5, size=16):
    """Bullet list with optional bold prefix. items = [(bold_part, rest_text), ...]"""
    txBox = slide.shapes.add_textbox(Inches(left), Inches(top), Inches(width), Inches(height))
    tf = txBox.text_frame; tf.word_wrap = True
    for i, (bold_part, rest) in enumerate(items):
        p = tf.paragraphs[0] if i == 0 else tf.add_paragraph()
        p.space_after = Pt(6)
        if bold_part:
            run = p.add_run(); run.text = bold_part
            run.font.size = Pt(size); run.font.bold = True; run.font.color.rgb = DARK_GRAY; run.font.name = "Calibri"
        run = p.add_run(); run.text = rest
        run.font.size = Pt(size); run.font.color.rgb = DARK_GRAY; run.font.name = "Calibri"

def add_table(slide, headers, rows, left=0.6, top=2.0, width=12, row_height=0.35):
    """Table with red headers and alternating row shading."""
    tbl_shape = slide.shapes.add_table(len(rows)+1, len(headers), Inches(left), Inches(top), Inches(width), Inches(row_height*(len(rows)+1)))
    table = tbl_shape.table
    for ci, h in enumerate(headers):
        cell = table.cell(0, ci); cell.text = h
        cell.fill.solid(); cell.fill.fore_color.rgb = RED
        for p in cell.text_frame.paragraphs:
            for r in p.runs: r.font.color.rgb = WHITE; r.font.bold = True; r.font.size = Pt(12); r.font.name = "Calibri"
        cell.vertical_anchor = MSO_ANCHOR.MIDDLE
    for ri, row_data in enumerate(rows):
        for ci, val in enumerate(row_data):
            cell = table.cell(ri+1, ci); cell.text = str(val)
            if ri % 2 == 1: cell.fill.solid(); cell.fill.fore_color.rgb = LIGHT_GRAY_BG
            for p in cell.text_frame.paragraphs:
                for r in p.runs: r.font.size = Pt(11); r.font.color.rgb = DARK_GRAY; r.font.name = "Calibri"
            cell.vertical_anchor = MSO_ANCHOR.MIDDLE
```

### Slide 5 Detail: Three-Column Team Layout

For the "What's In Flight" slide, use a 3-column layout instead of a table:

```python
col_width = 3.8; gap = 0.3; start_x = 0.6
for idx, (team, focus) in enumerate([("Forge", "xKS, GitOps & CI/CD"), ("Compass", "QE & Observability"), ("Heimdall", "Security & Gateway")]):
    x = start_x + idx * (col_width + gap)
    # Red header box with team name and count
    shape = slide.shapes.add_shape(MSO_SHAPE.RECTANGLE, Inches(x), Inches(1.2), Inches(col_width), Inches(0.5))
    shape.fill.solid(); shape.fill.fore_color.rgb = RED; shape.line.fill.background()
    # ... add team name in white, focus area in italic gray below, then top 4 items as bullet text
```

Prioritize items: strategic initiatives first, then blocker/critical priority, then remaining. Show item summary (truncated to 55 chars) with priority tag.

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
