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

The current format version is **6**.

| Version | Added |
|---|---|
| 1 | Initial format: `auth`, `metadataLookup`, `browse` (flat item listing), `tagsLookup`. |
| 2 | `browse.categories` (sidebar category sources, e.g. Jellyfin's libraries or a fixed facet like Tags/Genres), `browse.containerTypeValues` + `itemMapping.typePath` (container-vs-playable item detection for hierarchical browsing, e.g. Library → Series → Season → Episode). Also nested the REST item-listing fields (`listRequest`/`itemsPath`/`itemMapping`) under `browse.rest.listing` (a `RESTListingSpec`), shared with a category source's own listing — a shape change from v1's flat `browse.rest.{listRequest,itemsPath,itemMapping}`. `sortFieldMap`/`sortDirectionMap` (on a listing) and `staticEntries`/`dependsOnContextKey` (on a category source) were added to the interpreter alongside the Plex connector but were missing from this document until now — see [`itemMapping`](#itemmapping)-adjacent sort mapping and [Categorization and hierarchy](#categorization-and-hierarchy) below. They're additive and safely ignored by an older client (untranslated sort token, unnested category), so this doesn't bump the format version. |
| 3 | `browse.itemDetail` (`ItemDetailSpec`) — fetches full, current details for exactly one item by id, refreshing a Media-Library-opened video's metadata live rather than trusting a stale drag-time snapshot. See [Item detail](#browseitemdetail). |
| 4 | `metadataLookup.rest` (`RESTMetadataLookupSpec`) — title-search metadata lookup for a REST connector with no fingerprint API (e.g. a TMDB-/TheTVDB-style catalog). `metadataLookup.dialects` became optional as part of this (a REST-transport connector sets `rest` instead). See [`metadataLookup`](#metadatalookup). |
| 5 | `auth.headerValuePrefix` — a static prefix (e.g. `"Bearer "`) sent in front of the resolved secret under `headerName`, for a server whose header-based auth needs more than the bare secret. See [`auth`](#auth). |
| 6 | Array-mapping paths accept a filter bracket, `[key=value]` / `[key!=value]`, alongside `[]` — e.g. `People[Type=Actor].Name` keeps only the cast entries that are actors. Bumped (unlike a purely additive display field) because an older client can't interpret the bracket and would silently map *no* elements — an empty cast that looks like "none" — instead of refusing the connector. See [Path syntax reference](#path-syntax-reference). |

Ratings, favoriting, view-count, and several informational `itemMapping`
fields (cover image, codecs, bitrate, frame rate, resolution) were added to
the interpreter alongside the Plex connector but, like `sortFieldMap` in v2,
were missing from this document until now. They're additive and safely
ignored by an older client (no rating/favorite control shown, no extra
detail line), so none of them bump the format version on their own — see
[Ratings, favorites, and actions](#ratings-favorites-and-actions) and the
[`itemMapping`](#itemmapping) reference below. The top-level `attribution`
field (see [`attribution`](#attribution)) is the same kind of purely
additive, safely-ignored field.

## Top-level shape

```jsonc
{
  "id": "my-server",           // stable identifier, not shown to users
  "name": "My Server",         // shown in GridPlayer's UI
  "version": 6,
  "transport": "rest",         // "rest" | "graphql" — picks which of the two browse/lookup shapes below apply
  "auth": { ... },             // required
  "metadataLookup": { ... },   // optional — fingerprint- or title-search-based metadata matching
  "browse": { ... },           // optional — library browsing (either transport)
  "tagsLookup": { ... },       // optional — flat tag/category list for a filter picklist (GraphQL only, today)
  "actions": { ... },          // optional — write operations: rate, favorite, increment view count
  "attribution": { ... }       // optional — a required-credit notice some data sources' own terms mandate
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
  "headerName": "Authorization", // header the resolved credential is sent under (header/login)
  "extraHeaders": { ... },     // optional — static headers always sent, e.g. required client-ID headers
  "login": { ... },            // required when type == "login"
  "queryParamName": "apikey",  // optional, defaults to "apikey" — see below
  "headerValuePrefix": "Bearer " // optional (v5+) — see below
}
```

- **`header`**: a single static, pre-shared secret (an admin-issued API key)
  sent under `headerName`, optionally prefixed by `headerValuePrefix`.
- **`login`**: a username/password is exchanged for a session token via one
  request, then that token is sent the same way `header` would send a static
  one, `headerValuePrefix` included. See `login` below.
- **`none`**: no credential at all.

`headerValuePrefix` (v5+) is static text prepended to the resolved secret —
e.g. TMDB's API expects `Authorization: Bearer <token>`, not the bare token,
so a TMDB-style connector sets `"headerValuePrefix": "Bearer "`. `null`/
omitted (every connector before this field existed) sends the secret as-is.
Only affects the header form — there's no equivalent prefix convention for
the `queryParamName` form below.

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

Exactly one of `dialects` (GraphQL, hash-based) / `rest` (REST, title-search
based) is set, matching the connector's top-level `transport`.

### Hash-based (`metadataLookup.dialects`, GraphQL only)

Single-item metadata matching against a GraphQL server — "does this server
know a video with this file hash, and if so, what's its title, cast, tags,
chapters?"

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

### Title-search (`metadataLookup.rest`, v4+, REST only)

For a server with no fingerprint/hash API at all — a catalog you search by
title instead (TMDB, TheTVDB). GridPlayer derives a title and, where
possible, a release year from the video's file name (a best-effort
heuristic: dots/underscores as word separators, a four-digit year or an
`SxxEyy` episode marker ending the title), searches, and feeds the first
(highest-relevance) result's id into the *same* single-item detail fetch
`browse.itemDetail` uses — there's no separate metadata-only field-mapping
shape; `itemMapping` (`title`/`castPath`/`tagsPath`/`chapters`) already
covers everything a match needs, so a title-search connector's `itemMapping`
simply leaves every browsing-only field (`streamURLPath`, `durationPath`,
...) unset.

```jsonc
{
  "rest": {
    "searchRequest": { "method": "GET", "path": "search/movie", "query": { "query": "{title}", "year": "{year}" } },
    "resultsPath": "results",
    "resultIDPath": "id",
    "resultTypePath": null,
    "resultTypeMap": null,
    "detail": { "rest": { "request": { "method": "GET", "path": "movie/{id}", "query": null }, "itemMapping": { ... } } }
  }
}
```

- **`searchRequest`**: may reference `{title}` and, when the file name
  yielded one, `{year}` — an unresolved `{year}` is dropped like any other
  unresolved REST placeholder, so the template can reference it
  unconditionally.
- **`resultsPath`**: path to the search response's results array.
- **`resultIDPath`**: path, relative to the *first* result, to that result's
  own id — substituted into `detail`'s `{id}`. Accepts either a JSON string
  or a whole-number JSON value (TMDB's `id` is a bare integer, not a
  string).
- **`resultTypePath`** / **`resultTypeMap`** (both optional): for a server
  whose detail endpoint shape depends on the matched entity's kind — e.g.
  TheTVDB's `type` field (`"movie"`/`"series"`) picks between
  `movies/{id}/extended` and `series/{id}/extended`. `resultTypePath` reads
  the raw value from the first result; `resultTypeMap` translates it into
  whatever literal token `detail` actually needs before substituting
  `{type}` (`"movie"` doesn't mechanically pluralize into `"movies"`, and
  `"series"` must not be touched at all — a static lookup, not a string
  transformation rule, same rationale as `sortFieldMap`). `null` passes the
  raw value through unchanged. Omit both for a connector with only one
  entity kind (a movie-only catalog, say), which then never references
  `{type}` in `detail` at all.
- **`detail`**: an `ItemDetailSpec` — see [Item detail](#browseitemdetail)
  below for its shape. Reused as-is; nothing metadata-specific about it.

An empty results array, or a result whose id can't be resolved, is a normal
"no match" outcome (`null` metadata), not an error.

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
server with no chapters simply omits `chapters`. `id` accepts either a JSON
string or a whole-number JSON value (some REST APIs report a bare integer).

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
  "typePath": null,                 // optional — see Categorization and hierarchy
  "ratingPath": null,               // optional — see Ratings, favorites, and actions
  "ratingDivisor": null,
  "isFavoritePath": null,
  "coverURLPath": null,             // optional — higher-resolution image, distinct from screenshotURLPath
  "videoCodecPath": null,           // optional — purely informational, shown in file-info detail
  "audioCodecPath": null,
  "bitratePath": null,
  "bitrateDivisor": null,
  "frameRatePath": null,
  "widthPath": null,
  "heightPath": null,
  "viewCountPath": null             // optional — see Ratings, favorites, and actions
}
```

`coverURLPath`/`videoCodecPath`/`audioCodecPath`/`bitratePath`/
`bitrateDivisor`/`frameRatePath`/`widthPath`/`heightPath` are purely
informational — GridPlayer shows them in file-info detail but never
interprets them. `bitrateDivisor` follows the inverse convention from
`durationDivisor`/`ratingDivisor` (both "raw ÷ divisor"): it's a
**multiplier's reciprocal**, e.g. `0.001` if the server reports kbps instead
of bits/second, since a fractional divisor stands in for the multiplication
that direction of unit conversion needs.

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

### `browse.itemDetail` (v3+)

Fetches full, current details for exactly one item by id — used to refresh
a Media-Library-opened video's metadata live, rather than trusting a stale
drag-time snapshot or the browse listing's necessarily thinner per-item
mapping. It's also what a title-search `metadataLookup.rest` reuses for its
own detail step (see [Title-search](#title-search-metadatalookuprest-v4-rest-only)
above) — there's nothing metadata-specific about it. Exactly one of
`rest`/`graphql` is set, matching the connector's transport.

```jsonc
{
  "rest": {
    "request": { "method": "GET", "path": "items/{id}", "query": null },
    "itemPath": null,                 // optional — path to the item node if the response wraps it
    "itemMapping": { ... }            // same shape as browse.rest.listing.itemMapping
  }
}
```

or, for a GraphQL connector:

```jsonc
{
  "graphql": {
    "query": "query FindItem($id: ID!) { findItem(id: $id) { title } }",
    "variables": { "id": "{{id}}" },
    "resultPath": "findItem",
    "itemMapping": { ... }
  }
}
```

`itemPath` is `null`/omitted when the response body *is* the item (most
REST APIs' single-item endpoint); set it when the single-item endpoint
still wraps the result in the listing's own shape (e.g. Plex:
`"MediaContainer.Metadata[0]"`).

`null`/omitted entirely (every connector before this field existed) means
this capability doesn't exist for this connector — a video opened from the
Media Library then just keeps whatever came with it already.

## Ratings, favorites, and actions

Added to the interpreter alongside the Plex connector, without their own
format version bump (see [Versioning](#versioning)) — an older GridPlayer
build simply shows no rating/favorite control rather than misinterpreting
anything.

**Reading a rating or favorite flag** — on `itemMapping`:

```jsonc
{
  "ratingPath": "userRating",   // this server's own rating field
  "ratingDivisor": 2,           // normalizes to GridPlayer's 0–5 scale, e.g. Plex's 0–10 userRating ÷ 2
  "isFavoritePath": "isFavorite"
}
```

`ratingDivisor` follows the "raw ÷ divisor" convention (`null` means the raw
value is already 0–5). `isFavoritePath` is `null` for a connector with no
favorite concept at all — distinct from a resolved `false` ("known not
favorited").

**Filtering by rating or favorite in the toolbar** — on `browse`:

```jsonc
{
  "supportsFavoriteFilter": true,
  "maxRatingFilter": 5,                     // shows a 1...N star filter control
  "ratingFilterScale": 2,                   // scales the toolbar's raw star count up to this server's native range
  "ratingFilterExclusiveComparator": false, // true if this server's filter is strictly-greater-than only
  "ratingFilterModifierToken": null         // literal token for a companion {{minRatingModifier}} placeholder some GraphQL-style filters need
}
```

Both `null`/omitted (every connector before these fields existed) hides the
corresponding toolbar control entirely.

**Write operations** — top-level `actions` (`ActionsSpec`), each
independently optional; GridPlayer only shows a write control (a tappable
star, a heart button) when the corresponding action is present:

```jsonc
{
  "rate": { "rest": { "method": "PUT", "path": "items/{id}/rate/{value}", "query": null }, "graphql": null },
  "rateValueScale": 2,        // rescales GridPlayer's 0-5 star count to this server's native range before {value}/{{value}} substitution
  "addFavorite": { ... },
  "removeFavorite": { ... },  // split from addFavorite since the two directions are often genuinely different requests (e.g. POST vs. DELETE on the same path)
  "incrementViewCount": { ... } // no {value} needed, just {id}/{{id}}
}
```

Each action is an `ActionSpec` (`{ "rest": RESTRequestSpec, "graphql": null }`
or the reverse) — a REST one uses the usual `{placeholder}` substitution in
`path`/`query`; a GraphQL one (`{ "query": ..., "variables": { ... } }`)
substitutes `{{id}}`/`{{value}}` into `variables` the same way a read query
does, then prunes anything left unresolved.

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

## `attribution`

Some data sources' own terms require a visible credit and link wherever
their data is shown — not every server needs this (Jellyfin/Plex don't),
but a public catalog API (TMDB, TheTVDB) typically does. Added without a
format version bump: it's purely additive display data, safe for an older
GridPlayer build to just not show.

```jsonc
{
  "notice": "This product uses the Example API but is not endorsed or certified by Example.",
  "url": "https://example.com",
  "iconURL": "https://example.com/logo.png"   // optional
}
```

`notice` is shown verbatim — GridPlayer never rewords it, since it's the
source's own required wording, not descriptive copy. `null`/omitted (every
connector before this field existed) shows nothing extra.

`iconURL` (optional, also added without a version bump — an older client
just doesn't show it) is a logo shown next to the notice, for a source whose
terms ask for one. It must be an `https` URL to a raster image (PNG or JPEG
— SVG isn't rendered). GridPlayer fetches it with no credential, only while
the attribution is on screen, treats it as purely decorative (the `notice`
text is what carries the meaning), and shows just the text if it isn't
`https` or fails to load. Whether hotlinking a given logo is permitted is up
to the connector's author to check with the source.

## Path syntax reference

- **Dot path**: `"a.b.c"` — walks nested objects.
- **Array index**: `"a[0].b"` — a specific element.
- **Array mapping**: `"a[].b"` (used for `cast`/`tags`/`namesPath`/
  `arrayPath` contexts) — resolves the whole array, then `b` on every
  element. An empty head (`"[].b"`) means the node itself is already the
  array.
- **Array filter** (v6+): `"a[key=value].b"` / `"a[key!=value].b"` — like
  array mapping, but keeps only the elements whose `key` does (`=`) or
  doesn't (`!=`) equal `value`. `key` is a path relative to each element
  (`type`, or nested: `person.type`). The comparison is exact and
  case-sensitive against the field's text form — a string, a whole number, or
  `true`/`false`. An element missing the field never matches `=` and always
  matches `!=`. `value` runs to the closing `]`, so it can't contain `]` or
  `.`. Only the first bracket in a path filters; a bare index like `[0]` isn't
  an array bracket and is skipped. A filter with no key matches nothing.
  Example: Jellyfin-style `"People[Type=Actor].Name"` lists only the actors
  from a `People` array that also holds directors and writers.
- **GraphQL variable templates**: `"{{name}}"` — whole-value-only
  substitution inside a `variables` JSON tree.
- **REST path/query templates**: `"{name}"` — substring substitution inside
  a path or query-value string.

## Constraints

- A connector document is limited to 1&nbsp;MB when fetched from a URL.
- Nothing in this format executes code. It describes requests and response
  shapes only; GridPlayer's interpreter is fixed and pre-reviewed.
