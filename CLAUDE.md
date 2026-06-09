# Strategy Review Skill

Claude Code skill for generating data-backed AICP strategy review documents for leadership meetings.

## Available Skills

- **strategy-review** — Generate a data-backed strategy review document covering portfolio, deliverables, in-flight work, customer signal, and opportunities

## Available Commands

- `/strategy-review` — Invoke the strategy review skill

## Data Sources

1. **AICP Feature Priorities Spreadsheet** (Google Sheet) — Roadmap tab distinguishes owned vs support work
2. **Jira REST API v3** — 7 queries scoped to `component in ("AI Core Platform", "AI Core Platform Security")`

## Output

```
docs/strategy/YYYY-MM-DD_strategy_review.docx  # Strategy review Word doc
docs/strategy/YYYY-MM-DD_strategy_review.md     # Strategy review markdown
```
