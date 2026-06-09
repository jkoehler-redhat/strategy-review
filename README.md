# Strategy Review Skill

A Claude Code skill that generates data-backed strategy review documents for AI Core Platform (AICP) leadership meetings. It pulls live data from two sources, categorizes work by ownership and sub-team, and produces a formatted Word document ready for director-level review.

## What It Does

Generates a 7-section strategy review document:

1. **What We Own** — Roadmap features split by ownership (AICP-owned vs cross-team support), plus engineering portfolio breakdown by sub-team
2. **What We've Delivered** — Resolved issues grouped by theme (xKS, Gateway, Observability, Security, etc.) over a configurable lookback period
3. **What's In Flight** — Active issues by sub-team with key items, priority, and assignee
4. **Customer Signal** — Customer-labeled issues from RHOAIENG and RHAISTRAT, with honest reporting when direct customer labels are zero
5. **Where We See Opportunity** — Synthesized analysis of unassigned work, ownership gaps, and strategic alignment
6. **What We Need** — Capacity asks, blocker escalations, and process needs for leadership
7. **Next Steps** — Action/owner/timeline table derived from blockers and open items

## Data Sources

### 1. AICP Feature Priorities Spreadsheet (Google Sheet)

The Roadmap tab is the source of truth for what AICP owns vs supports:

- **Ownership = "AICP"** — Features this team owns and drives
- **Ownership = "Support"** — Features another team owns that AICP reviews, consults on, or does supporting engineering for

Exported via `gws drive files export` and parsed with Python stdlib (`zipfile` + `xml.etree.ElementTree` — no pip install needed).

### 2. Jira REST API v3

Seven queries, all scoped to `component in ("AI Core Platform", "AI Core Platform Security")`:

| # | Query | Purpose |
|---|-------|---------|
| 1 | All open issues | Portfolio total |
| 2 | Status in (In Progress, In Review, In Testing, ...) | In-flight by sub-team |
| 3 | Resolved in lookback period | Delivered, grouped by theme |
| 4 | Customer labels on RHOAIENG | Direct customer signal |
| 5 | Customer/field labels on RHAISTRAT | Strategic customer signal |
| 6 | Blocker/Critical priority, open | Blockers snapshot |
| 7 | In-flight without `aicp-team-*` label | Unassigned gap |

Uses cursor-based pagination (`nextPageToken`/`isLast`). Does **not** use Jira MCP tools — all queries go through the REST API directly.

## How It Works

### Step 1: Interactive Prompts

The skill asks three questions before collecting data:

- **Lookback period** — How far back to look for delivered work (default: 90 days)
- **Output format** — Word document, markdown, or both (default: Word)
- **Additional context** — Optional strategic notes, meeting outcomes, or leadership asks

### Step 2: Data Collection

1. Exports the Google Sheet as `.xlsx` and parses the Roadmap tab
2. Extracts Jira credentials from `~/.claude.json` (mcpServers config)
3. Runs all 7 Jira queries via REST API, paginating to get complete results
4. Categorizes issues by sub-team label (`aicp-team-forge`, `aicp-team-compass`, `aicp-team-heimdall`)
5. Groups delivered issues by theme using keyword matching

### Step 3: Document Generation

Generates a Word document using `python-docx` with:

- Red Hat red (#CC0000) table headers with white text
- Alternating row shading (#F2F2F2)
- Thin black borders on all table cells
- Section headers in Red Hat red, body text in 11pt Calibri

### Step 4: Output

Saves to `docs/strategy/YYYY-MM-DD_strategy_review.docx` (and/or `.md`), then asks the user for revisions.

## Usage

```
/strategy-review
```

Or ask Claude Code for a "strategy review", "leadership review", or "portfolio review".

## Prerequisites

- **Claude Code** with access to this skill
- **Google Workspace CLI** (`gws`) — for exporting the AICP Feature Priorities spreadsheet
- **Jira API token** — configured in `~/.claude.json` under `mcpServers.mcp-atlassian`
- **python-docx** — for Word document generation

## Sub-Teams

| Team | Label | Focus Area |
|------|-------|------------|
| Forge | `aicp-team-forge` | xKS Expansion, GitOps & CI/CD |
| Compass | `aicp-team-compass` | E2E Stability, QE & Observability |
| Heimdall | `aicp-team-heimdall` | Security, Gateway & Authentication |
