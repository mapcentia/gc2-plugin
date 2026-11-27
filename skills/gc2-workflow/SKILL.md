---
name: gc2-workflow
description: Use when turning versioning or workflow on or off for a table in GC2 BaaS, giving sub-users a workflow role (author, reviewer, publisher), listing or approving drafted/reviewed features, or reading the workflow log — the versioning/workflow fields on getTable/patchTable, the getWorkflowRoles/patchWorkflowRoles/getWorkflowItem/patchWorkflowItem/getWorkflowLog tools, gc2_version_* / gc2_status / gc2_workflow columns, or VERSIONING_REQUIRED / WORKFLOW_ENABLED / NO_WORKFLOW_ROLE / WORKFLOW_STATUS_CONFLICT / HSTORE_NOT_AVAILABLE errors.
---

# Versioning and Workflow

Two per-table switches, set on the table resource, plus the Workflow API for roles, approval and the log.

- **Versioning**: edits through WFS-T and the Feature API keep the previous state of a feature as an *ended version* (`gc2_version_*` columns).
- **Workflow** (needs versioning): every feature has a status — `1` drafted, `2` reviewed, `3` published. A sub-user's workflow role decides the status their edits get.

Editing features is `gc2-feature` (Feature API) and `gc2-ows` (WFS-T); table privileges are `gc2-privileges`.

## Availability

On GeoCloud2 local `master` as of 2026-11-27 — **not pushed, not deployed, not on production `api.gc2.io`**. Check the target host's OpenAPI first. MCP tools ship in `@mapcentia/gc2-mcp-server` 1.2.13+.

## Switching on and off (`getTable` / `patchTable`)

`getTable` returns `versioning` and `workflow` (booleans) in the full response — not in the `namesOnly` summary. `patchTable` accepts both.

| Request | Result |
|---|---|
| `versioning: true` | Adds the `gc2_version_*` columns |
| `workflow: true` | Adds `gc2_status` and `gc2_workflow`; existing rows are marked published. Needs versioning — both may be enabled in one request |
| `versioning: false` | **Deletes every ended version and drops the columns — the history is lost** |
| `workflow: false` | **Drops `gc2_status`/`gc2_workflow` and the table's workflow log**; every row is an ordinary row afterwards, whatever status it had |

Errors:

| Status / code | Cause |
|---|---|
| `403 SUPER_USER_ONLY` | Only the database owner may set either field |
| `409 NOT_A_TABLE` | Views and other relations cannot be versioned |
| `409 VERSIONING_REQUIRED` | `workflow: true` on an unversioned table without `versioning: true` in the same request |
| `409 WORKFLOW_ENABLED` | `versioning: false` while workflow stays on — disable workflow in the same request |
| `501 HSTORE_NOT_AVAILABLE` | Enabling workflow on a database that has not been migrated (the `hstore` extension is outside the search path). Newly created databases are fine |

**Disabling either one is destructive.** Before sending `versioning: false` or `workflow: false`, say what will be lost and get explicit confirmation (`gc2-rules` section 5).

## Roles

`getWorkflowRoles` / `patchWorkflowRoles` — `/api/v4/schemas/{schema}/tables/{table}/workflow-roles`.

| Role | Effect |
|---|---|
| `author` | Edits become drafts (status 1) |
| `reviewer` | Edits and approvals count as reviewed (status 2) |
| `publisher` | Edits and approvals are published (status 3) |
| `none` | No part in the workflow: sees published features only and cannot edit the table |

- GET lists only sub-users that have been given a role: `[{ "subuser": "joe", "role": "author" }]`.
- PATCH takes one `{subuser, role}` object or an array and **merges** — sub-users not named keep their role. Answers `303`. Database owner only (`403 SUPER_USER_ONLY`); unknown table → `404 TABLE_NOT_FOUND`.
- The database owner needs no role and approves as publisher.
- Access to the Workflow API is by workflow role, not by schema ownership.

## Items and approval

`/api/v4/schemas/{schema}/tables/{table}/workflow-items/{gid}` — `{gid}` is the feature's primary key value.

**`getWorkflowItem`**

- Without `gid`: the items the caller can still move forward, newest first — drafted features for a reviewer, drafted and reviewed ones for a publisher and the database owner. `all=true` lists every item.
- With `gid` (or a comma list): that item whatever its status.
- Item: `{ schema, table, gid, version_gid, status, status_name, operation, user, author, reviewer, publisher, created }`. `operation` is `insert`, `update` or `delete`; `author`/`reviewer`/`publisher` hold who signed, or `null`; `version_gid` is the same across all versions of the feature.
- `403 NO_WORKFLOW_ROLE`; `404 TABLE_NOT_FOUND` / `WORKFLOW_ITEM_NOT_FOUND`.

**`patchWorkflowItem`** — approve. **No body.**

- Signs the feature(s) with the caller's role and moves them to that role's status. The state before the step is kept as an ended version.
- `gid` may be a comma list: all or nothing.
- Answers `303`.

| Status / code | Cause |
|---|---|
| `403 NO_WORKFLOW_ROLE` | Caller has no role on the table |
| `404 TABLE_NOT_FOUND` / `FEATURE_NOT_FOUND` | Unknown table or feature |
| `409 WORKFLOW_STATUS_CONFLICT` | The feature is already at or past the caller's status |
| `409 VERSION_ENDED` | The id addresses an ended version |
| `409 WORKFLOW_NOT_ENABLED` | The table has no workflow |

There is no reject or "send back" operation in the API; a feature only moves forward by approval, or gets a new status from a new edit.

## The log across tables

`getWorkflowLog` — `GET /api/v4/workflow-log`: every step logged by an edit or an approval, newest first.

- Query: `schema` (only that schema's tables), `limit` (default 100, 1–1000), `offset`.
- Entry: the item fields plus `id` and `roles` (the sub-user → role map on the table when the step was made). `status` is the status that step gave the feature.
- The database owner gets the whole log; a sub-user the entries of tables they have a role on; no role anywhere → `[]`.
- To send an entry's feature on, call `patchWorkflowItem` with the entry's `schema`, `table` and `gid`.

## Common mistakes

| Mistake | Reality |
|---|---|
| Sending `versioning: false` to "tidy up" | Deletes the whole history. Confirm first |
| Enabling workflow without versioning | `409 VERSIONING_REQUIRED`; send both as `true` in one request |
| Disabling versioning on a workflow table | `409 WORKFLOW_ENABLED`; send `workflow: false` too — and both losses apply |
| Sending a body or a status to `patchWorkflowItem` | It takes no body; the caller's role decides the status |
| Approving a feature twice, or as author after review | `409 WORKFLOW_STATUS_CONFLICT` |
| Expecting schema ownership or `write` privilege to give access to items | The workflow role decides; without one → `403 NO_WORKFLOW_ROLE` |
| Sub-user "cannot see" new features | Without a role they read published features only |
| Looking for `versioning`/`workflow` in `getSchema`/`namesOnly` output | Only in the full `getTable` response |
| Setting the switches with `postTable` | The tool lists the fields (shared `Table` schema), but only `patchTable` sets them — create the table, then PATCH |
| Assuming production availability | Local `master` only as of 2026-11-27 |
