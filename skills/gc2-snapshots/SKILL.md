---
name: gc2-snapshots
description: Use when exporting GC2 BaaS tables/views as Parquet or FlatGeobuf snapshots, or reading published snapshots for analytics — /api/v4/snapshots job endpoints, /api/v4/schemas/{schema}/relations/{relation}/snapshots read endpoints, /api/v4/snapshots/catalog/rebuilds (rewriting the STAC catalog), the Snapshot tools, DuckDB/read_parquet over HTTP Range, presigned URLs, snapshot formats, or snapshot_date-based downloads.
---

# Snapshots (Parquet/FlatGeobuf exports)

Use this skill for the v4 Snapshots APIs: asynchronous exports of a relation to object storage (job API, `/api/v4/snapshots`) and the read API for published snapshots (`/api/v4/schemas/{schema}/relations/{relation}/snapshots`).

## Availability and access

On GC2 local master as of 2026-09 — **not** on production `api.gc2.io`. MCP tools ship with `@mapcentia/gc2-mcp-server` 1.1.0+ (server restart required); no `@mapcentia/gc2-sdk` class as of 0.3.0. **Job API is super-user only** (403 otherwise; 501 `SNAPSHOT_NOT_CONFIGURED` if the instance has no snapshot storage config). **Read API allows sub-users** — by schema ownership or layer read privilege (403 `INSUFFICIENT_PRIVILEGES`), but **never a sub-user a deny/limit geofence rule applies to** (403 `GEOFENCE_RULES_APPLY` — a whole-table file cannot honour a row filter; allow-rules do not block).

## Job API (queue and poll)

| Operation | Route / MCP tool | Result |
|---|---|---|
| Queue | `POST /api/v4/snapshots`, body `{schema, relation, srs?, formats?}` **or an array** of them / `postSnapshot` | `202` `{id, status: "pending", _links.self}` (array in → array out). Arrays are **all-or-nothing**: every relation is checked first (`404` `RELATION_NOT_FOUND`, `409` `SNAPSHOT_IN_PROGRESS`, `400` duplicate/empty) |
| Poll | `GET .../snapshots/{id}` / `getSnapshot` — one id or a comma list | `200` job; `404` `NO_SNAPSHOT_ERROR` |
| List jobs | `GET .../snapshots?schema=&relation=` / `getSnapshots` | `200`, the 50 newest job rows (bare array) |

A cron worker (`snapshot_worker.php`) runs the export with ogr2ogr — **poll until `status` is `succeeded` or `failed`** (`pending`/`running` in between; `superseded` = a later job re-exported the same relation+date and replaced this one's publication). `formats` (ordered, non-empty, no duplicates; enum `parquet`/`flatgeobuf`; omitted = server default): a geometry-needing format on a geometry-less relation is **skipped with a reason** — unless *every* requested format needs geometry, which is refused up front (400). Per-format outcomes land in `formats[]` (`SnapshotFormatResult`: `format`, `status` requested/produced/skipped, `file`, `size_bytes`, `media_type`, `reason`, `href`).

## STAC catalog rebuild (repair only)

Each database's snapshots are described by a static STAC catalog in the snapshot storage (`catalog.json`, per relation `collection.json` + `latest.json`, per snapshot `item.json`). **A publish updates only its own relation's documents plus `catalog.json`** — so there is normally nothing to do. Queue a full rewrite only to repair a catalog that has drifted (missing/stale documents of other relations) or after a server upgrade that changes the document layout.

| Operation | Route / MCP tool | Result |
|---|---|---|
| Queue | `POST /api/v4/snapshots/catalog/rebuilds`, **no body** / `postSnapshotCatalogRebuild` | `202` `{id, status: "pending", _links.self}`; `409` `CATALOG_REBUILD_IN_PROGRESS` while one is pending or running; `501` `SNAPSHOT_NOT_CONFIGURED` |
| Poll | `GET .../catalog/rebuilds/{id}` / `getSnapshotCatalogRebuild` — one id or a comma list | `200` `{id, status, documents, failed_documents, error, username, created, started, finished}`; `404` `CATALOG_REBUILD_NOT_FOUND` |
| List | `GET .../catalog/rebuilds` / `getSnapshotCatalogRebuilds` | `200`, the 50 newest (bare array) |

Super-user only. MCP tools ship with `@mapcentia/gc2-mcp-server` 1.2.6+, SDK methods (`Snapshots.postSnapshotCatalogRebuild()` etc.) with `@mapcentia/gc2-sdk` 0.3.4+. `status` is `pending`/`running`/`succeeded`/`failed`; `failed` with `failed_documents > 0` means the rest were written. **It is slow on a large catalog** (one storage write per document — minutes for thousands of snapshots) and the same worker runs snapshots, so queued snapshots on that server wait while it runs. On a server that predates the endpoint the route does not exist; there every publish still rewrites the whole catalog.

## Read API (published snapshots)

- `getRelationSnapshots` — bare array, newest first: `{snapshot_date, snapshot_id, row_count, size_bytes, schema_version, files[], formats[], published}`.
- `getRelationSnapshot` — one by `{date}` (`YYYY-MM-DD`, **or `latest`** for the newest published snapshot — a fixed URL; the response still carries the real `snapshot_date`, and `_links.latest` points there); adds `srs`, `relation_schema`, `crs`, `_links`. `400` malformed date, `404` `NO_SNAPSHOT_ERROR`. All four `{date}` routes (metadata + the three file routes) accept `latest`.
- Three file routes, all with HEAD + **single** byte-range support (206/416, ETag): `/data` = the primary file (the Parquet when produced, else the only produced file; `409` `MULTI_FILE_SNAPSHOT` when several files and no Parquet — the message lists the per-format URLs); `/data/{format}` = a named format (`400` unknown, `404` skipped/not produced — check `formats[]` for a `produced` entry first); `/files/{name}` = a raw catalog file by name (`data-<id>.<ext>`, `metadata-<id>.json` — the only route to the metadata sidecar). Storage failure → `502` `SNAPSHOT_STORAGE_ERROR`.

**The MCP tools for the file routes (`getRelationSnapshotData`, `getRelationSnapshotDataFormat`, `getRelationSnapshotFile`) never return the binary body** — they HEAD the route and answer `{url, presigned_url?, content_type, size_bytes, etag, accept_ranges, note}`. Read the `url` out-of-band with the same bearer token; in redirect mode the server answers `302` and the short-lived `presigned_url` is fetched **without** auth headers (don't cache it).

## Analytics pattern (DuckDB)

```python
url = ".../relations/bygninger/snapshots/latest/data"        # fixed URL, newest published snapshot
con.execute("CREATE SECRET (TYPE http, BEARER_TOKEN '<token>');")
con.execute(f"SELECT count(*) FROM read_parquet('{url}')")   # httpfs does HEAD + Range reads
```

Pin a specific date (`.../snapshots/2026-09-22/data`) when the analysis must not drift as new snapshots publish.

Scheduler jobs can queue snapshots automatically after each successful import (`snapshot: true`, `snapshot_formats`) — see `gc2-scheduler`.

## Common mistakes

| Mistake | Reality |
|---|---|
| Assuming production availability | GC2 local master only as of 2026-09; MCP tools need `@mapcentia/gc2-mcp-server` 1.1.0+ + restart |
| Queueing with a sub-user token | Job API is super-user only → `403`; only the read API allows sub-users |
| Treating the `202` as "exported" | It queues a job — poll `getSnapshot` until `succeeded`/`failed` |
| Expecting file bytes from the data tools | They return URL + HEAD metadata; fetch the URL with DuckDB/requests |
| Sending auth headers to a `presigned_url` | It is presigned — fetch it bare, and never cache it |
| `/data/{format}` for a skipped format | `404` — check the snapshot's `formats[]` for `status: "produced"` first |
| Geofenced sub-user reading snapshots | Deny/limit rules → `403` `GEOFENCE_RULES_APPLY` regardless of read privilege |
| Queueing a catalog rebuild after every snapshot | Not needed — a publish already updates its own documents; a rebuild is a slow repair job that holds up queued snapshots |
| Partial batch queueing | Arrays are all-or-nothing — one bad relation fails the whole POST |
