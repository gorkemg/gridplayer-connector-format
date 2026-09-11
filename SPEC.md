# GridPlayer Connector Format

A connector is a JSON document that tells GridPlayer how to talk to one kind
of media server: which HTTP or GraphQL requests to send, how to authenticate,
and how to map the response onto GridPlayer's own data (titles, cast, tags,
chapters, browsable items). It contains no executable code — GridPlayer's
built-in interpreter reads it and makes the requests itself. Writing a
connector means describing your server's API in this format; it does not
require touching GridPlayer's own source.

GridPlayer ships with exactly one connector built in (Jellyfin —
[`examples/jellyfin.json`](examples/jellyfin.json)). Every other connector is
loaded from a URL you provide, in the app's server settings.

## Versioning

`version` is an integer, required to be `>= 1`. It exists so an older
GridPlayer build can refuse a connector that uses a capability it doesn't
understand yet, instead of silently misinterpreting it. Bump it only when
your connector needs a *capability* the interpreter doesn't have — a new
auth flow, a new field-mapping shape. Targeting a different server, a
different query, or different field names never requires a version bump;
the interpreter is generic over those already.

The current format version is **2**.

| Version | Added |
|---|---|
| 1 | Initial format: `auth`, `metadataLookup`, `browse` (flat item listing), `tagsLookup`. |
| 2 | `browse.categories` (sidebar category sources, e.g. Jellyfin's libraries or a fixed facet like Tags/Genres), `browse.containerTypeValues` + `itemMapping.typePath` (container-vs-playable item detection for hierarchical browsing, e.g. Library → Series → Season → Episode). Also nested the REST item-listing fields (`listRequest`/`itemsPath`/`itemMapping`) under `browse.rest.listing` (a `RESTListingSpec`), shared with a category source's own listing — a shape change from v1's flat `browse.rest.{listRequest,itemsPath,itemMapping}`. `sortFieldMap`/`sortDirectionMap` (on a listing) and `staticEntries`/`dependsOnContextKey` (on a category source) were added to the interpreter alongside the Plex connector but were missing from this document until now — see [`itemMapping`](#itemmapping)-adjacent sort mapping and [Categorization and hierarchy](#categorization-and-hierarchy) below. They're additive and safely ignored by an older client (untranslated sort token, unnested category), so this doesn't bump the format version. |

## Top-level shape

```jsonc
{
  "id": "my-server",           // stable identifier, not shown to users
  "name": "My Server",         // shown in GridPlayer's UI
  "version": 2,
  "transport": "rest",         // "rest" | "graphql" — picks which of the two browse/lookup shapes below apply
  "auth": { ... },             // required
  "metadataLookup": { ... },   // optional — hash-based metadata matching (GraphQL only, today)
  "browse": { ... },           // optional — library browsing (either transport)
  "tagsLookup": { ... }        // optional — flat tag/category list for a filter picklist (GraphQL only, today)
}
```

A connector needs at least one of `metadataLookup` / `browse` to be useful,
but the format doesn't require both — a server that only does hash-based
matching (no personal library to browse) can omit `browse` entirely, and vice
versa.

## `auth`

```jsonc
{
  "type": "header",            // "header" | "login" | "none"
  "headerName": "ApiKey",      // header the resolved credential is sent under (header/login)
  "extraHeaders": { ... },     // optional — static headers always sent, e.g. required client-ID headers
  "login": { ... },            // required when type == "login"
  "queryParamName": "apikey"   // optional, defaults to "apikey" — see below
}
```

- **`header`**: a single static, pre-shared secret (an admin-issued API key)
  sent as-is under `headerName`.
- **`login`**: a username/password is exchanged for a session token via one
  request, then that token is sent the same way `header` would send a static
  one. See `login` below.
- **`none`**: no credential at all.

`queryParamName` matters for anything that can't send a custom header —
`AVPlayerItem(url:)` for streaming, a plain image download for a screenshot.
GridPlayer appends the credential as a query parameter on those URLs instead,
under whatever name your server expects (`apikey`, `api_key`, ...). It's only
ever appended when the resolved URL's host matches the server's own
configured host — never to a URL on a different host, even one your
connector's own request templates point at.

### `login`

```jsonc
{
  "path": "auth/login",           // POST path, relative to the server's base URL
  "usernameField": "username",    // JSON body field name for the username
  "passwordField": "password",    // JSON body field name for the password
  "tokenResponsePath": "token"    // path into the login response where the session token lives
}
```

## `metadataLookup`

Hash-based, single-item metadata matching against a GraphQL server — "does
this server know a video with this file hash, and if so, what's its title,
cast, tags, chapters?"

```jsonc
{
  "dialects": [
    {
      "id": "my-dialect",
      "query": "query FindByHash($input: HashInput!) { findByHash(input: $input) { title } }",
      "variables": { "input": { "hash": "{{hash}}" } },
      "resultPath": "findByHash",
      "mapping": { "title": "title", "cast": "performers[].name", "tags": "tags[].name", "chapters": null }
    }
  ]
}
```

`dialects` is tried in order; the first one the server accepts (doesn't
respond with an "unknown field/argument/type" GraphQL error) wins. This lets
one connector cover multiple schema variants of the same underlying software
without GridPlayer needing to know which one it's talking to.

`variables` is a template: any string leaf written as `"{{hash}}"` is
replaced with the actual file hash before the request is sent — GraphQL
variables, so this is safe against injection (the substituted value is sent
as structured JSON data, never spliced into the query text).

`mapping.cast` / `mapping.tags` use **array-mapping path** syntax:
`"performers[].name"` means "resolve `performers` as an array, then `name`
on each element." `mapping.chapters` (optional) works the same way over a
nested array — `arrayPath` locates it, `secondsPath` is required per
element, everything else is optional.

## `browse`

Paginated item listing plus how to resolve a stream URL. Exactly one of
`rest` / `graphql` is set, matching the connector's top-level `transport`.
Two more fields, both optional, add sidebar categorization and hierarchical
drill-down on top — see [Categorization and hierarchy](#categorization-and-hierarchy)
below.

### REST (`browse.rest`)

```jsonc
{
  "listing": {
    "listRequest": { "method": "GET", "path": "items", "query": { "q": "{searchText}" } },
    "itemsPath": "items",
    "itemMapping": { ... }          // see below
  },
  "streamRequest": { "method": "GET", "path": "items/{id}/stream", "query": null },
  "imageRequest": { "method": "GET", "path": "items/{id}/image", "query": null }  // optional
}
```

`listing` (a `RESTListingSpec`: `listRequest` + `itemsPath` + `itemMapping`,
plus the optional `sortFieldMap`/`sortDirectionMap` below) is the same shape
reused by a category source's own listing (see below) — one "request a list,
map each item" building block for both.

#### Sort mapping (`sortFieldMap` / `sortDirectionMap`)

Both optional, on a `RESTListingSpec` and on `browse.graphql` alike. GridPlayer
sorts by canonical tokens (`"name"` / `"date"` / `"size"` for the field,
`"asc"` / `"desc"` for the direction) and substitutes them into
`{sortField}`/`{sortDirection}` (REST) or `{{sortField}}`/`{{sortDirection}}`
(GraphQL). `sortFieldMap`/`sortDirectionMap` translate those canonical tokens
into whatever your server's own API expects before substitution — e.g.
Jellyfin's field map is `{"name": "SortName", "date": "DateCreated"}` and its
direction map is `{"asc": "Ascending", "desc": "Descending"}`, while Plex
accepts `"asc"`/`"desc"` directly and needs no direction map at all.

```jsonc
{
  "sortFieldMap": { "name": "SortName", "date": "DateCreated" },
  "sortDirectionMap": { "asc": "Ascending", "desc": "Descending" }
}
```

A canonical token missing from the map (including when the map itself is
`null`) is left as an unresolved placeholder, which is then dropped from the
request entirely — the same "drop unresolved placeholders" rule as REST
path/query substitution. That's how a server that can't sort along some axis
(Plex has no file-size sort) simply omits that request parameter instead of
sending broken literal text.

`imageRequest` is only needed when `itemMapping.screenshotURLPath` is `nil`
— some servers (Jellyfin included) never return an image URL as a field,
only an item id, and expect the client to build the image URL itself the
same way it builds the stream URL.

`path`/`query` values use **`{placeholder}`** substitution (single braces,
unlike the double-brace GraphQL variable templates above) from context that
includes the current item's own `id`, the search text, page/perPage, sort
field/direction, plus any REST path placeholders your server needs (e.g.
`{userId}`, `{parentId}`). A query value whose placeholder never got resolved
(e.g. `{parentId}` when nothing is selected yet) is dropped from the request
entirely rather than sent as literal, unresolved text — so one template
covers every hierarchy depth, including the unscoped root.

### GraphQL (`browse.graphql`)

```jsonc
{
  "query": "query List($filter: FilterType) { items(filter: $filter) { count nodes { id title } } }",
  "variables": { "filter": { "q": "{{searchText}}", "page": "{{page}}", "per_page": "{{perPage}}" } },
  "resultPath": "items",
  "itemsPath": "nodes",
  "totalCountPath": "count",
  "itemMapping": { ... }
}
```

### `itemMapping`

Field mapping for one browsable item, relative to that item's own JSON node.
Only `id` and `title` are required — everything else is best-effort; a
server with no chapters simply omits `chapters`.

```jsonc
{
  "id": "id",
  "title": "title",
  "durationPath": "duration",       // optional
  "durationDivisor": null,          // optional — e.g. 10000000 if the server reports 100ns ticks, not seconds
  "fileSizePath": null,
  "screenshotURLPath": null,        // omit + use browse.rest.imageRequest if the server has no image field
  "streamURLPath": null,            // omit + use browse.rest.streamRequest similarly
  "castPath": "performers[].name",
  "tagsPath": "tags[].name",
  "chapters": null,
  "typePath": null                  // optional — see Categorization and hierarchy
}
```

## Categorization and hierarchy

Two optional `browse` fields turn a single flat item list into a sidebar
tree with drill-down — entirely opt-in; a connector that omits both behaves
exactly as in format version 1, a single flat browsable source.

### `browse.categories: [CategorySourceSpec]`

Each entry is one sidebar category source:

```jsonc
{
  "label": "Libraries",
  "expandInSidebar": true,        // see below
  "rest": { ... },                // a RESTListingSpec, same shape as browse.rest.listing — exactly one of rest/graphql is set
  "graphql": null,
  "selectionContextKey": "parentId"
}
```

- **`expandInSidebar: true`** (e.g. Jellyfin's libraries): this source's own
  listing results become the sidebar tree's children directly, one row per
  item — no extra click needed to see them. Selecting a row binds its id to
  `selectionContextKey` and opens the connector's *normal* item listing
  (`browse.rest`/`browse.graphql`) scoped to it.
- **`expandInSidebar: false`** (a fixed facet, e.g. a "Tags" or "Genres"
  browse entry whose actual values live server-side): the sidebar shows one
  fixed row, `label` (e.g. "Tags") — no fetch needed to know it exists.
  Selecting it runs *this source's own* listing to show the actual facet
  values as containers; selecting one of those then binds
  `selectionContextKey` (e.g. `"tagId"`) into the normal item listing.

Two more fields on `CategorySourceSpec`, both optional:

- **`staticEntries: [{ id, title }]`** — fixed sidebar rows needing no
  network fetch at all, e.g. a single "All Scenes" entry with no facet
  filter. When set, `rest`/`graphql` are ignored for producing sidebar rows
  (they may still be `null`). Each entry's `id` binds to
  `selectionContextKey` when selected, exactly like a dynamically-fetched
  entry would — except an empty `id` (`""`) is a sentinel meaning "no filter
  at all," since no real server-assigned id is ever empty.
- **`dependsOnContextKey: String`** — for a source whose own listing must be
  scoped to a value chosen from *another* category first, e.g. Plex's
  `library/sections/{sectionId}/actor` needs a library already selected,
  unlike a globally-listable facet such as Jellyfin's global people. When
  set, this source is never shown at the sidebar root; instead
  it appears nested under every *resolved* entry of whichever other source's
  `selectionContextKey` matches this value, and that entry's bound id is
  merged into this source's own listing request. Omit it (every source
  without this dependency) for a source that behaves exactly as always, at
  the top level.

### `browse.containerTypeValues: [String]` + `itemMapping.typePath`

`typePath` points at a field on each item carrying a server-native
type/kind string (e.g. Jellyfin's `"Type"`, values `"Series"`/`"Season"`/
`"Movie"`/`"Episode"`). Any item whose resolved type value is listed in
`containerTypeValues` is a still-drillable container rather than a playable
leaf — selecting it re-runs the *same* item listing with its id bound to
`selectionContextKey`, one level deeper. This is how an arbitrarily deep
hierarchy (Library → Series → Season → Episode) is expressed with a single
generic recursive mechanism rather than fixed levels: `containerTypeValues`
applies uniformly to every listing this connector produces, so the same
check terminates the recursion at whatever depth the server's own data
actually bottoms out.

A connector with no hierarchy concept (a flat scene list, say) omits
`typePath` — every item is then always a leaf, and `containerTypeValues`
is irrelevant.

## `tagsLookup`

Lists every known tag/category the server has, for a filter picklist,
independent of any specific item — GraphQL only, today.

```jsonc
{
  "query": "query AllTags { allTags { name } }",
  "resultPath": "allTags",
  "namesPath": "[].name"
}
```

## Path syntax reference

- **Dot path**: `"a.b.c"` — walks nested objects.
- **Array index**: `"a[0].b"` — a specific element.
- **Array mapping**: `"a[].b"` (used for `cast`/`tags`/`namesPath`/
  `arrayPath` contexts) — resolves the whole array, then `b` on every
  element. An empty head (`"[].b"`) means the node itself is already the
  array.
- **GraphQL variable templates**: `"{{name}}"` — whole-value-only
  substitution inside a `variables` JSON tree.
- **REST path/query templates**: `"{name}"` — substring substitution inside
  a path or query-value string.

## Constraints

- A connector document is limited to 1&nbsp;MB when fetched from a URL.
- Nothing in this format executes code. It describes requests and response
  shapes only; GridPlayer's interpreter is fixed and pre-reviewed.
