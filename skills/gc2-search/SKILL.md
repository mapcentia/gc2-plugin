---
name: gc2-search
description: Use when adding or debugging full-text search over a table or view in GC2 BaaS — typeahead/autocomplete, "search as you type", OpenSearch/Elasticsearch, /api/v4/schemas/{schema}/tables/{table}/search, /api/v4/search/settings, the search/searchPost/buildSearchIndex/dropSearchIndex/getSearchSettings/putSearchSettings tools, the Search SDK class, per-column mapping (elasticsearchtype, analyzer, search_analyzer) in Meta, or INDEX_NOT_FOUND / INDEX_BUILD_ERROR / SEARCH_ERROR / NOT_OWNER errors.
---

# Search (OpenSearch index per relation)

Use this skill for the v4 Search API: each table or view can have an OpenSearch index, built from its rows on request and queried with the OpenSearch `_search` syntax. The index is **a copy, not a live view** — it holds the rows as they were at the last build.

## Availability

MCP tools are generated from `centia-api.json`; SDK support is `@mapcentia/gc2-sdk` **0.3.5+** (`Search` class). Whether the API is deployed on a given server — including production `api.gc2.io` — is not confirmed here: a `404` on `/api/v4/search/settings` means the server does not have it.

## Endpoints

Bearer token on every request. Searching is open to sub-users with **read access to the relation**; building, dropping, the mapping and the analysis settings are for the **database owner only** (`403` `NOT_OWNER`).

| Operation | Route / MCP tool | Result |
|---|---|---|
| Search by query string | `GET /api/v4/schemas/{schema}/tables/{table}/search` / `search` | `200` OpenSearch response |
| Search by query DSL | `POST` same route, body = `_search` body / `searchPost` | `200` OpenSearch response |
| (Re)build the index | `PUT` same route, **no body** / `buildSearchIndex` | `200` `{index, message, errors, errors_in}` |
| Drop the index | `DELETE` same route / `dropSearchIndex` | `200` `{index, dropped}` — the table is untouched |
| Read analysis settings | `GET /api/v4/search/settings` / `getSearchSettings` | `200` `{analysis: {...}}` (the default when none is set) |
| Set analysis settings | `PUT /api/v4/search/settings`, body `{analysis: {...}}` / `putSearchSettings` | `200`, echoes the stored block |

`{table}` takes a table **or a view**. The index is named `<database>_<schema>_<table>`.

## Building the index

`buildSearchIndex` drops any existing index, creates it with the database's analysis settings and a mapping derived from the columns, bulk indexes every row and refreshes — so it is **searchable as soon as the call returns**. It is synchronous: there is no job to poll.

- The relation **must have a primary key** (it becomes `_id`); otherwise `400` `INDEX_BUILD_ERROR`.
- Check `errors` on the result: `true` means the index exists but some documents failed, listed in `errors_in`. Show `message` ("Indexed N documents") to the user either way.
- Rebuild after the data changes, after a mapping change and after an analysis change — none of them reach an existing index on their own.

## Documents and field names

Rows are indexed as GeoJSON-like documents: `_source` is `{properties: {<column>: value, ...}, geometry: ...}`. **Every attribute is addressed as `properties.<column>`** — in `q`, in the DSL, in `sort` and in `_source` filters. A bare column name matches nothing.

## Searching

`search` passes the query string to OpenSearch URI search unaltered, so any `_search` URI parameter works: `q` (Lucene syntax), `size` (default 10), `from`, `sort` (`properties.name.keyword:asc`), `_source` (comma list, or `false`), `default_operator` (`AND`/`OR`), `df`. With no parameters every document matches.

`searchPost` passes the body to `_search` unaltered: `query`, `size`, `from`, `sort`, `_source`, `aggs`, `highlight`, and anything else OpenSearch accepts.

```json
{"query": {"match": {"properties.name": "Findme"}}, "sort": [{"properties.name.keyword": "asc"}], "size": 20}
```

The response is the raw OpenSearch one: `hits.total.value`, `hits.hits[]` with `_id`, `_score`, `_source`, and `aggregations` when the query has `aggs`. `_score` is `null` when sorting on something other than the score.

- **`default_operator` and `df` are only valid together with `q`.** Sent alone, OpenSearch answers `400` ("contains unrecognized parameter") — leave them out for a match-all.
- Sort and aggregate text on the **`.keyword`** sub-field (`properties.name.keyword`), not on the text field itself.
- `403` on a search comes in two kinds for sub-users. `INSUFFICIENT_PRIVILEGES`: no read access to the relation (see `gc2-privileges`). `GEOFENCE_RULES_APPLY`: a deny or limit rule applies to the user, and search is refused outright rather than filtered — **retrying gives the same answer**, so tell the user search is unavailable for that table.
- `404` `INDEX_NOT_FOUND` means the relation exists but **no index has been built** — build it (owner) rather than treating it as a missing table. A sub-user cannot build it: to them search is simply not available.
- `400` `SEARCH_ERROR` is a query OpenSearch rejected; its message says why.

## Per-column mapping

The build derives each column's mapping from its PostgreSQL type: text → `text` with a `keyword` sub-field, integers → `integer`/`long`, decimals → `float`/`double`, dates → `date`, point geometry → `geo_point`, other geometry → `geo_shape`. Overrides are laid on top; they live in the relation's Meta, in `elasticsearch`, keyed by column name.

Read them with `getMetaData` (`relations["schema.table"].elasticsearch` — absent when there are none; `.fields` lists the columns) and write them with `patchMetaData`:

```json
{"relations": {"my_schema.poi": {"elasticsearch": {
  "name": {"elasticsearchtype": "text", "analyzer": "auto_complete_analyzer", "search_analyzer": "auto_complete_search_analyzer"},
  "category": {"elasticsearchtype": "keyword"}
}}}}
```

Per-column keys, all optional — leave out what is unset: `elasticsearchtype` (`text`, `keyword`, `integer`, `long`, `float`, `double`, `boolean`, `date`, `geo_point`, `geo_shape`, `ip`), `analyzer` and `search_analyzer` (text only), `index` (boolean; `false` = not searchable), `format` (dates), `null_value`, `fielddata` (boolean), `normalizer`, and `fields` (multi-fields).

- **PATCH replaces the relation's whole `elasticsearch` block.** Always send every column that has overrides, not just the one you changed — the others are lost otherwise. Read, modify, write back.
- Reset one column by sending the block without it; clear everything with `"elasticsearch": {}`.
- Setting `fields` replaces the automatic `keyword` sub-field, so `.keyword` disappears unless you include it yourself (`{"keyword": {"type": "keyword", "ignore_above": 256}}`).
- Old overrides from the v2 GUI may carry `id`, `column` and `type`; the build ignores them — keep them when round-tripping. Their `elasticsearchtype: "string"` and non-boolean `index` values are invalid in current OpenSearch: use `text`/`keyword` and `true`/`false`.
- `patchMetaData` answers `303` with a `Location` header (see `gc2-provisioning`).

## Analysis settings and autocomplete

`getSearchSettings` returns the per-database `settings.analysis` block (`analyzer`, `tokenizer`, `filter`, `char_filter`, `normalizer`). `putSearchSettings` **replaces it wholesale** — read it, add to it, write it all back — and it applies only to indexes built afterwards. A body without an `analysis` object is `400` `INVALID_DATA`.

An analyzer named in a mapping must exist: either a key in `analysis.analyzer` or an OpenSearch built-in (`standard`, `simple`, `whitespace`, `keyword`, `stop`, language analyzers such as `danish` and `english`). An unknown one fails the build with `400` `INDEX_BUILD_ERROR`.

The default block ships `auto_complete_analyzer` (index time, edge n-grams) and `auto_complete_search_analyzer` (search time). Prefix search — "Fin" finding "Findme" — is those two on a text column, then a rebuild:

1. `patchMetaData` with `analyzer: "auto_complete_analyzer"` and `search_analyzer: "auto_complete_search_analyzer"` on the column (plus every other override already there).
2. `buildSearchIndex`.
3. `search` with `q=properties.name:Fin`.

## SDK (0.3.5+)

Explicit-client pattern: `new Search(client)` with a `createCentiaClient(...)` instance, or `new Search(adminClient.http)`. Methods mirror the tools: `search(schema, table, params?)`, `searchPost(schema, table, body)`, `buildSearchIndex(schema, table)`, `dropSearchIndex(schema, table)`, `getSearchSettings()`, `putSearchSettings({analysis})`. Type the hits with `SearchSource<{name: string}>`: `searchPost<SearchSource<Row>>(...)` gives `hit._source.properties.name`. Mapping overrides go through `adminClient.provisioning.metadata.patchMetaData`.

Errors are `CentiaApiError`. The API sends the machine-readable code as `errorCode` next to a numeric `code`; from 0.3.5 the SDK exposes it as `error.code` (`"INDEX_NOT_FOUND"`), and the raw body is in `error.details`.

## Common mistakes

| Mistake | Reality |
|---|---|
| Querying `name:foo` | Attributes live under `properties.` — `properties.name:foo` |
| Expecting new rows to be searchable | The index is a copy; rebuild after data changes |
| Changing a mapping or analyzer and searching again | Nothing changes until `buildSearchIndex` runs |
| Reading `404` `INDEX_NOT_FOUND` as a missing table | The relation exists; its index has not been built |
| PATCHing only the column you changed | PATCH replaces the whole `elasticsearch` block — send every column with overrides |
| PUTting one new analyzer to `/search/settings` | The block is replaced wholesale; read, extend, write back |
| Sending `default_operator` without `q` | OpenSearch rejects it (`400`); only send it with a query string |
| Sorting on a text field | Sort on `properties.<column>.keyword` |
| Setting `fields` and still using `.keyword` | `fields` replaces the automatic sub-field; include `keyword` yourself |
| Polling for the build to finish | The build is synchronous; the index is searchable when `PUT` returns |
| Building an index on a relation without a primary key | `400` `INDEX_BUILD_ERROR` — add a primary key first |
| Retrying a search that answered `403` `GEOFENCE_RULES_APPLY` | An access rule covers the user; search is refused, not filtered — the answer will not change |
| Having a sub-user build or drop | Owner only: `403` `NOT_OWNER`. Sub-users can search relations they can read |
