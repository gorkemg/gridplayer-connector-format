# Contributing a connector

1. Read [`SPEC.md`](SPEC.md).
2. Write your connector as `examples/<server-name>.json`, lowercase,
   hyphenated (e.g. `examples/my-server.json`).
3. Validate it against the schema before opening a PR:

   ```bash
   npx ajv-cli validate -s schema/connector.schema.json -d examples/<server-name>.json
   ```

   The schema catches structural mistakes (missing required fields, wrong
   types) but can't verify your queries/paths actually match your server's
   real API — test that against a live instance yourself.
4. Open a PR. Include, in the description:
   - what server/software this targets, and a link to its own API docs if
     it has any
   - which capabilities you've tested (metadata lookup, browsing, login)
   - the GridPlayer version you tested against

## A connector is data, not code

Nothing in this format executes. If your server needs something the current
format can't express — a new auth flow, a response shape the interpreter
can't map — open an issue describing the gap rather than trying to work
around it; that likely means a format version bump, which needs a
GridPlayer-side change too.

## Scope

This repository documents a generic request/response description format and
hosts connectors for it. It is not a directory or endorsement of any
particular server software, and pull requests should describe connectors in
purely technical terms (protocol, schema, fields) rather than promotional
ones.
