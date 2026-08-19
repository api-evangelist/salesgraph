---
name: Query the OMS graph and set up a continuous research watch
description: Use Salesgraph's OMS (Opportunity Management System) endpoints to find an Account or Opportunity, traverse its relationships, inspect where its data came from, and then put a cost-capped continuous research watch on it.
api: openapi/salesgraph-oms-api-openapi.yml
operations: [getOmsMetadata, searchOmsObjects, getOmsObject, pivotOmsObject, getOmsProvenance, createOmsWatch, listOmsWatches, cancelOmsWatch]
mcp_tools: [oms_metadata, oms_search, oms_get, oms_pivot, oms_provenance, oms_watch_create, oms_watch_list, oms_watch_get, oms_watch_cancel]
---

# Query the OMS graph and watch an account

The OMS endpoints read your organization's own sales objects — not public data. They are the
only Salesgraph surface that returns **JSON** rather than markdown, and everything they return is
scoped to the organization the API key belongs to.

## Auth

Send the key on every request as `Authorization: Bearer sg_live_...` (or `x-api-key`). A missing,
malformed, or revoked key returns `401` with `{ "error": "unauthorized" }` and a
`WWW-Authenticate: Bearer` header.

## Steps

1. **Learn the ontology first.** Call `getOmsMetadata` (`GET /api/v1/oms/metadata`). It returns
   the object types, links, and properties *this* organization can see. Do not assume a type or
   link name — the visible ontology varies by org and by which integrations are connected.
2. **Find the object.** Call `searchOmsObjects` (`POST /api/v1/oms/search`) with the `type` from
   step 1 and a `query`. The response is `{ objects, nextPageToken }`; pass `nextPageToken` back
   as `pageToken` to page, and stop when it is `null`.
3. **Read one object.** Call `getOmsObject` (`POST /api/v1/oms/get`) with `type` and the
   **public** `key` — the `domain:acme.com` form, never an internal primary key. An internal id
   is the single most common cause of a `not_found` here.
4. **Traverse.** Call `pivotOmsObject` (`POST /api/v1/oms/pivot`) with `type`, `key`, `link`, and
   `direction` to walk a relationship (Account to its Opportunities, for example). Same
   `{ objects, nextPageToken }` page shape as search.
5. **Check lineage before you act on it.** Call `getOmsProvenance`
   (`POST /api/v1/oms/provenance`) with the same `type`/`key`. It returns `properties`,
   `relationships`, and `history` with the source each value came from — use it to say *why* a
   value is what it is before an agent writes anything downstream.
6. **Create the watch.** Call `createOmsWatch` (`POST /api/v1/oms/watches`) with:
   - `target`: `{ "type": "Account" | "Opportunity", "key": "<public key>" }`
   - `query`: what to monitor, e.g. `"Acme funding and leadership changes"`
   - `frequency`: `1h` through 30 days
   - `processor`: `lite` or `base`
   - `monthlyCostCapMicros`: at least `3000` for `lite`, at least `10000` for `base`, at most
     `100000000` for either
   - `sourcePolicy` (optional): up to 25 valid domains per `includeDomains`/`excludeDomains`
     list, and no domain in both lists
   - `idempotencyKey`: **always send one.** It is the only idempotency contract Salesgraph
     publishes. A retry carrying the same key will not create a duplicate watch; a retry without
     one will.
7. **Manage it.** `listOmsWatches` (`GET /api/v1/oms/watches`) returns
   `{ watches, nextCursor }` — note the cursor field here is `nextCursor`, not the
   `nextPageToken` the object endpoints use. Pass `?id=<watch-id>` to read one.
   `cancelOmsWatch` (`DELETE /api/v1/oms/watches?id=<watch-id>`) stops it.

## Requesting a write

OMS never lets an agent write directly. `requestOmsAction` (`POST /api/v1/oms/actions`) creates a
**human approval request** — an opportunity update, a rep note, or a drafted follow-up email —
and returns `202` with the request. Nothing is applied until a person approves it. Treat the
`202` as "queued for a human", not as success.

## Errors

OMS input, access, and missing-resource failures all use the flat JSON envelope:
`{ "error": "invalid_input" | "access_denied" | "not_found" }`. See
`errors/salesgraph-problem-types.yml`.

## Gotchas

- Public keys only (`domain:acme.com`), never internal primary keys.
- Cost caps are **monthly** and are enforced per processor tier — a `lite` watch with a cap below
  3000 micros is rejected as `invalid_input`.
- A domain may not appear in both the include and exclude lists.
- Paging fields differ between the two families: `nextPageToken` on objects, `nextCursor` on
  watches.
