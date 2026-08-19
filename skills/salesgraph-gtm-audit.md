---
name: Run a GTM audit of a prospect and poll for the result
description: Use Salesgraph to start an org-aware go-to-market audit of a prospect's website, then poll the async run until the full markdown audit is ready.
api: openapi/salesgraph-commands-api-openapi.yml
operations: [runCommand, pollRun, startOrgAudit, pollOrgAudit]
mcp_tools: [gtm_audit, org_audit, get_run_status]
---

# Run a GTM audit and poll for the result

GTM and org audits are **long-running** (tens of seconds to several minutes). They start
immediately and return a run id you poll — never block a tight loop on them.

## Auth
Send your API key on every request as `Authorization: Bearer sg_live_...`. Audits require a
completed org sales profile; if onboarding is not finished, the call returns an
`Onboarding required` message (HTTP `428` on REST) instead of starting a run.

## Steps — prospect GTM audit

1. **Start the audit.** Call `runCommand` with `command = gtm-audit` and body `{ "website": "<domain-or-url>" }`.
   - REST: `POST /api/v1/commands/gtm-audit` with `{ "website": "acme.com" }`.
   - MCP: call tool `gtm_audit` with `{ "website": "acme.com" }`.
   - Returns `202` with the run id in the `X-Run-Id` header and `X-Run-Status: running`.
2. **Poll the run.** Call `pollRun` with `kind = gtm-audit` and the returned `id`.
   - REST: `GET /api/v1/runs/gtm-audit/<id>`.
   - MCP: call tool `get_run_status` with `{ "kind": "gtm-audit", "id": "<id>" }`.
   - Poll every 10-20 seconds. The `X-Run-Status` header carries `running` / `completed` / `failed`.
3. **Finish.** When status is `completed`, the poll returns the full audit as markdown. Stop polling on `completed` or `failed`.

## Steps — your own org audit

- Start with `startOrgAudit` (`POST /api/v1/audit`, or MCP tool `org_audit` with no arguments).
  Poll with `kind = audit` (`get_run_status` `{ "kind": "audit", "id": "<id>" }`) or `GET /api/v1/audit?id=<id>`.
- To block until done, pass `{ "wait": true }` (or `?wait=true`) — note this can return `504` if it exceeds the server timeout; the run keeps going, so poll its id.

## Rules
- Use the correct `kind` (`gtm-audit` vs `audit`) for the run — a mismatched kind returns `not found`.
- Run ids are scoped to your org; polling another org's id returns `not found` (`404`).
- Success bodies are `text/markdown`; errors are JSON `{ "error": "<code>" }`.
