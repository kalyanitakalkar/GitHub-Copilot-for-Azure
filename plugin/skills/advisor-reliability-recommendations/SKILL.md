---
name: advisor-reliability-recommendations
description: "Fetch and prioritize a user's active Azure Advisor reliability recommendations. Queries Azure Resource Graph via ARM MCP for active recos, then enriches them with priority scores via Advisor MCP, and presents them ranked by priority. WHEN: \"prioritize reliability recommendations\", \"advisor reliability recommendations\", \"azure advisor\", \"reliability recommendations\", \"priority score\", \"high availability recommendations\", \"which recommendations first\"."
license: MIT
metadata:
  author: Microsoft
  version: "0.0.0-placeholder"
---

# Azure Advisor Reliability Recommendation Prioritization

Fetch the user's active Azure Advisor recommendations and present them **ranked by priority
score**. Reliability (`HighAvailability`) is the default category.

## Quick Reference

| Property | Details |
|---|---|
| Best for | Prioritizing active Advisor reliability recommendations across subscriptions |
| Approach | ARM MCP fetches active recos (ARG) → Advisor MCP enriches with priority scores |
| Category | HighAvailability (default); Cost, Performance, Security, OperationalExcellence |
| MCP servers | `azure-arm-mcp`, `advisor-mcp-dev` |

## When to Use This Skill

Activate when the user wants to:
- "Prioritize my Azure Advisor reliability recommendations"
- "Which reliability recommendations should I fix first?"
- "Show my high-availability Advisor findings ranked by priority"

## Prerequisites

- User is logged in to Azure (ARM MCP uses the user's own credentials)
- Reader access on the target subscriptions

## MCP Tools

| Server | Tool | Key parameters |
|--------|------|----------------|
| `azure-arm-mcp` | `execute_query` | `subscriptions` (all IDs), `query` (KQL) |
| `advisor-mcp-dev` | `fetch_prioritized_recommendations` | `argResultsJson`, `topN` (10), `language` (`en`) |

> ⚠️ **Warning:** Use `advisor-mcp-dev` for enrichment — never `advisor-local` or any other Advisor server.

## Workflow

Run these steps **in order** (full KQL, filters, pagination, and output format in
[references/arg-query.md](references/arg-query.md)):

1. **Scope** — Present the user's available subscriptions as a selectable list and let them choose which to analyze. **The user may select at most 10 subscription IDs at once** — if more are chosen, ask them to narrow the selection to 10 or fewer. Also capture optional filters (category, subcategories, resource groups, retirement window).
2. **Query ARG (ARM MCP)** — Call `execute_query` **once** with the selected subscription IDs (≤ 10) in `subscriptions`. Follow `$skipToken` pagination; combine pages into one JSON array.
3. **Enrich (Advisor MCP)** — Pass the **raw** Step 2 JSON to `fetch_prioritized_recommendations` (`argResultsJson`, `topN`, `language`).
4. **Present** — Render a table ranked by priority (`#`, Recommendation, Priority score/label, Impact, Resources, Benefit); highlight Critical/High and suggest next steps.

## Error Handling

| Condition | Remediation |
|-----------|-------------|
| ARG returns no rows | Tell the user there are no active recos for that category; suggest another scope |
| More than 10 subscriptions selected | Ask the user to reduce the selection to 10 or fewer before querying |
| Invalid ARG JSON in Advisor MCP | Pass the raw, unmodified `execute_query` output |
| Not authenticated | Ensure the user is logged in to Azure |

> 💡 Never split into per-subscription ARG queries — a single `summarize` gives the combined view.
