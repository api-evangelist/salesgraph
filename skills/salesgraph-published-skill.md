---
name: Salesgraph
description: Use when agents need to research companies, map competitors, audit GTM strategies, or query sales data. Reach for this skill when building AI agents that need real-time market intelligence, prospect analysis, or organization-aware audits scoped to your sales context.
metadata:
    mintlify-proj: salesgraph
    version: "1.0"
---

# SalesGraph Skill

## Product summary

SalesGraph is an MCP (Model Context Protocol) server that exposes go-to-market research and audit tools to any MCP-compatible client or agent. It provides public company research, competitor mapping, GTM audits of prospect websites, and org-aware audits of your own organization. The server runs at `https://salesgraph.com/api/mcp` and authenticates with API keys prefixed `sg_live_`. All responses are markdown designed for LLM readers. Tools are either synchronous (research, competitors) or asynchronous (GTM audit, org audit), with async runs polled via `get_run_status`. See the [primary docs](https://docs.salesgraph.com) for the complete reference.

## When to use

Reach for SalesGraph when:
- An agent needs to research a company, market, or topic from public sources
- You need a sourced list of a company's direct competitors
- You're building a GTM audit workflow for prospect websites scoped to your org's context
- You need to audit your own organization's go-to-market positioning
- An agent should discover what integrations (CRM, call recorder) are active for your org
- You're querying your organization's sales data (OMS — Opportunity Management System)
- You need to set up continuous research watches on accounts or opportunities

Do not use SalesGraph for: authentication, account management, dashboard operations, or tasks that don't require external research or audit capabilities.

## Quick reference

### API key management
- Create keys in dashboard: Settings → API Keys
- Keys are prefixed `sg_live_` and shown only once at creation
- Store securely; treat like a password
- Use separate keys per environment (production, testing)
- Revoke immediately if leaked; revocation is instant

### Authentication
Send on every request using either header:
```
Authorization: Bearer sg_live_xxxxxxxxxxxx
```
or
```
x-api-key: sg_live_xxxxxxxxxxxx
```

### Core tools (sync)
| Tool | Input | Returns | Use for |
|------|-------|---------|---------|
| `research` | `topic` (string) | Markdown brief with citations | Research any company, market, or question |
| `competitors` | `company` (string) | Markdown list with sources | Map direct competitors of a company |
| `help` | — | Markdown command catalog | Discover available tools at runtime |
| `list_capabilities` | — | Markdown list | Check which integrations are active |

### Core tools (async)
| Tool | Input | Returns | Poll with |
|------|-------|---------|-----------|
| `gtm_audit` | `website` (domain or URL) | Run id | `get_run_status` with `kind: "gtm-audit"` |
| `org_audit` | — | Run id | `get_run_status` with `kind: "audit"` |

### Polling async runs
```json
{
  "name": "get_run_status",
  "arguments": { "kind": "gtm-audit", "id": "<id>" }
}
```
- Poll every 10–20 seconds; avoid tight loops
- Stop when `status` is `completed` or `failed`
- Run ids are org-scoped; wrong org returns `not found`

### OMS tools (data queries)
| Tool | Purpose |
|------|---------|
| `oms_metadata` | Get visible OMS ontology metadata |
| `oms_search` | Search visible OMS objects by type and query |
| `oms_get` | Fetch one OMS object by type and key |
| `oms_pivot` | Traverse OMS relationships |
| `oms_provenance` | Get object history and data sources |
| `oms_watch_create` | Create continuous research watch on Account or Opportunity |
| `oms_watch_list` | List your research watches |
| `oms_watch_cancel` | Cancel a watch by id |
| `oms_request_update_opportunity` | Request human-approved opportunity update |
| `oms_request_log_rep_note` | Request human-approved rep note |
| `oms_request_draft_followup_email` | Request human-approved email draft |

### REST API endpoints
Base URL: `https://salesgraph.com/api/v1`

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/commands` | List command catalog |
| `POST` | `/commands/{command}` | Run a command (research, competitors, gtm-audit, audit) |
| `GET` | `/runs/{kind}/{id}` | Poll async run status |
| `POST` | `/audit` | Start org audit (or `?wait=true` to block) |
| `GET` | `/oms/metadata` | OMS ontology metadata |
| `POST` | `/oms/search`, `/get`, `/pivot`, `/provenance` | Query OMS data |
| `GET`, `POST`, `DELETE` | `/oms/watches` | Manage research watches |

## Decision guidance

### When to use sync vs. async tools

| Scenario | Use | Why |
|----------|-----|-----|
| Quick research on a company or market | `research` (sync) | Returns in tens of seconds; no polling needed |
| Find competitors of a company | `competitors` (sync) | Synchronous; returns immediately |
| Audit a prospect's website in detail | `gtm_audit` (async) | Takes minutes; returns run id to poll |
| Audit your own org's GTM | `org_audit` (async) | Takes minutes; requires completed onboarding |

### When to use MCP vs. REST API

| Scenario | Use | Why |
|----------|-----|-----|
| Building an agent in Claude, Cursor, or custom MCP client | MCP server at `https://salesgraph.com/api/mcp` | Native tool discovery, streaming, SDK support |
| Simple scripts or non-MCP HTTP clients | REST API at `https://salesgraph.com/api/v1` | Plain HTTP; no MCP overhead |
| Testing or debugging | Raw HTTP with curl | Direct JSON-RPC 2.0 inspection |

### When to use OMS tools

| Scenario | Use | Why |
|----------|-----|-----|
| Query your org's accounts or opportunities | `oms_search`, `oms_get` | Direct access to your sales data |
| Set up continuous monitoring of an account | `oms_watch_create` | Continuous research watch with cost cap |
| Traverse relationships (e.g., account → opportunities) | `oms_pivot` | Navigate OMS graph structure |
| Understand data lineage | `oms_provenance` | See history and sources of an object |

## Workflow

### 1. Research a company or topic
1. Call `research` with a specific question (e.g., "What does Stripe do and who are its customers?")
2. Receive markdown brief with inline citations
3. Use the brief in your agent's reasoning or response

### 2. Map competitors
1. Call `competitors` with a company name or domain
2. Receive markdown list with competitor names, domains, and one-line differentiation
3. Each entry is sourced

### 3. Run a GTM audit on a prospect
1. Call `gtm_audit` with the prospect's domain (e.g., `acme.com`)
2. Receive a run id and polling instruction
3. Poll `get_run_status` with `kind: "gtm-audit"` and the id every 10–20 seconds
4. When `status: completed`, receive full audit markdown (slides)
5. Stop polling once complete or failed

### 4. Audit your own organization
1. Ensure your org has completed onboarding (sales profile set up in dashboard)
2. Call `org_audit` with no arguments
3. Receive a run id and polling instruction
4. Poll `get_run_status` with `kind: "audit"` and the id every 10–20 seconds
5. When `status: completed`, receive full audit markdown (report)

### 5. Query your sales data (OMS)
1. Call `oms_search` with object type (e.g., `Account`) and a query
2. Receive paginated list of matching objects
3. For a single object, call `oms_get` with type and public key (e.g., `domain:acme.com`)
4. To traverse relationships, call `oms_pivot` with type, key, link name, and direction
5. To see data lineage, call `oms_provenance` with type and key

### 6. Set up a research watch
1. Call `oms_watch_create` with:
   - `target`: Account or Opportunity (type and public key)
   - `query`: What to monitor (e.g., "funding and leadership changes")
   - `frequency`: `1h` through 30 days
   - `processor`: `lite` or `base` (affects cost cap)
   - `monthlyCostCapMicros`: 3000–100000000 (lite) or 10000–100000000 (base)
   - `sourcePolicy`: Optional domain include/exclude list (max 25 per list, no overlap)
2. Watch runs continuously; poll `oms_watch_list` to see status
3. Cancel with `oms_watch_cancel` when done

## Common gotchas

- **Wrong `kind` for polling**: Polling a `gtm-audit` id with `kind: "audit"` returns `not found`. Always match the kind to the tool that started the run.
- **Tight polling loops**: Polling every millisecond wastes resources. Poll every 10–20 seconds instead.
- **Onboarding required**: Audit tools fail with `Onboarding required` if your org hasn't completed its sales profile. Finish onboarding in the dashboard first.
- **Key leaked**: If an API key is exposed, revoke it immediately from Settings → API Keys. Revocation is instant.
- **Org isolation**: Run ids and OMS data are scoped to your org. Polling an id from another org returns `not found`.
- **Async runs timeout**: GTM and org audits can take several minutes. Don't assume they complete in seconds.
- **Public data only**: `research` and `competitors` use public sources only; they don't read your CRM or call data. Use OMS tools for internal data.
- **OMS key format**: Use public OMS keys (e.g., `domain:acme.com`), not internal primary keys.
- **Watch cost caps**: Lite watches require a cap of at least 3000 micros; base watches require at least 10000. Caps are monthly.
- **Source policy domains**: Max 25 domains per include or exclude list; a domain cannot appear in both.

## Verification checklist

Before submitting work with SalesGraph:

- [ ] API key is stored securely and not committed to source control
- [ ] Authentication header is `Authorization: Bearer <key>` (or `x-api-key`)
- [ ] For sync tools (`research`, `competitors`), verified the markdown response is well-formed
- [ ] For async tools (`gtm_audit`, `org_audit`), confirmed polling uses the correct `kind` (`gtm-audit` or `audit`)
- [ ] Polling interval is 10–20 seconds, not a tight loop
- [ ] Polling stops once `status` is `completed` or `failed`
- [ ] Org has completed onboarding if using audit tools (check dashboard)
- [ ] OMS queries use public keys (e.g., `domain:acme.com`), not internal ids
- [ ] Watch cost caps are within the range for the processor type (lite: 3000–100000000, base: 10000–100000000)
- [ ] Source policy domains don't overlap and don't exceed 25 per list
- [ ] Error responses are handled (401 for auth, 428 for onboarding, 429 for rate limits)

## Resources

- **Full page navigation**: [https://docs.salesgraph.com/llms.txt](https://docs.salesgraph.com/llms.txt)
- **Tools reference**: [https://docs.salesgraph.com/tools/overview](https://docs.salesgraph.com/tools/overview)
- **Async runs and polling**: [https://docs.salesgraph.com/reference/async-runs](https://docs.salesgraph.com/reference/async-runs)
- **REST API reference**: [https://docs.salesgraph.com/reference/rest-api](https://docs.salesgraph.com/reference/rest-api)

---

> For additional documentation and navigation, see: https://docs.salesgraph.com/llms.txt