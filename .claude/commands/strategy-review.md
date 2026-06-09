Generate a strategy review document for AICP leadership meetings.

Use the strategy-review skill to:

1. Ask for lookback period (default: 90 days), output format (docx/markdown/both), and any strategic context
2. Query Jira REST API for all AICP-scoped issues (component = "AI Core Platform")
3. Categorize by sub-team (Forge, Compass, Heimdall) and theme
4. Identify customer signal from RHOAIENG and RHAISTRAT
5. Generate a 7-section document: What We Own, Delivered, In Flight, Customer Signal, Opportunities, What We Need, Next Steps
6. Save to docs/strategy/ directory

Default: 90-day lookback, Word document output.
