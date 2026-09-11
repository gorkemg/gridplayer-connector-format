# GridPlayer Connector Format

[GridPlayer](https://gridplayer.app) is a macOS video player. Beyond local
files, it can match metadata (titles, cast, tags, chapters) and browse a
library from a self-hosted media server — described declaratively, in JSON,
rather than hardcoded per server.

This repository documents that format and hosts community-contributed
connector definitions, so support for a given server doesn't require an
app update, a review from us, or even changing GridPlayer's own source at
all. A connector is pure data: request shapes, field-mapping paths, nothing
executable.

- [`SPEC.md`](SPEC.md) — the format, in full.
- [`examples/`](examples) — working connector definitions, starting with
  [Jellyfin](examples/jellyfin.json) (also the one connector GridPlayer
  ships built in).
- [`schema/connector.schema.json`](schema/connector.schema.json) — a JSON
  Schema you can validate a connector against before submitting it, with no
  GridPlayer install required.

## Using a connector

In GridPlayer's server settings, add a server and point its "Connector URL"
field at the raw URL of a connector JSON file (this repository's `examples/`
files work directly via their GitHub raw URL). No other setup needed.

## Contributing a connector

See [`CONTRIBUTING.md`](CONTRIBUTING.md).
