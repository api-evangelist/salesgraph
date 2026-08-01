---
name: Research a company and map its competitors
description: Use Salesgraph to produce a cited research brief on a company or topic and a sourced list of its direct competitors.
api: openapi/salesgraph-openapi.yml
operations: [runCommand]
mcp_tools: [research, competitors]
---

# Research a company and map its competitors

Salesgraph exposes synchronous GTM research over both an MCP server and a REST API. Both
`research` and `competitors` return cited markdown immediately (typically within tens of seconds).

## Auth
Send your API key on every request as `Authorization: Bearer sg_live_...` (or `x-api-key: sg_live_...`).
Keys are per-organization and are created at Settings -> API Keys in the dashboard.

## Steps

1. **Research the company.** Call `runCommand` with path `command = research` and body `{ "topic": "<company or topic>" }`.
   - REST: `POST /api/v1/commands/research` with `{ "topic": "Stripe" }`.
   - MCP: call tool `research` with `{ "topic": "Stripe" }`.
   - Returns `200` with a cited markdown brief.
2. **Map competitors.** Call `runCommand` with path `command = competitors` and body `{ "company": "<company>" }`.
   - REST: `POST /api/v1/commands/competitors` with `{ "company": "Notion" }`.
   - MCP: call tool `competitors` with `{ "company": "Notion" }`.
   - Returns `200` with a sourced markdown list of direct competitors.
3. **Optionally pass extra context.** Include `userContext` (string) in the body to steer the output.

## Rules
- Success bodies are `text/markdown`; errors are JSON `{ "error": "<code>" }`.
- `401` means a missing/invalid/revoked key — check the Authorization header.
- `429` means the org rate limit was hit — back off and retry.
- These commands are synchronous; do not poll runs for them.
