# CID — project conventions

CID (Central Intelligence Database) is a GreyScript database library for Grey
Hack. It's being built as the database layer for a separate project called
Smoke (a spy-themed hacking tool), but stays standalone and plug-and-play so
it can be dropped into any GreyScript project. Inspired by tuonux's BinDB —
the previous implementation is preserved on the `original_base` branch for
reference only; it does not follow the conventions below.

These are locked-in decisions for this repo. Follow them in every PR unless
the user explicitly changes one.

## Architecture

- The whole library lives in one file: `cid.src`. Single-file on purpose, so
  it's trivial to `import_code` into any project.
- One external dependency: `uuid.src` (a separate file, provided by the user),
  used to generate unique row IDs. No other dependencies.

## Code style

- `snake_case` for variables and functions.
- `SCREAMING_SNAKE_CASE` for constants.
- `PascalCase` for classes (prototype maps instantiated with `new`).
- Always call functions with `()`, even where GreyScript allows omitting them.
  Exception: `.len`, `.indexes`, `.values` and similar map/list intrinsics are
  properties, not functions — no parentheses on those.
- 8 spaces per indentation level.
- Keep functions short. Avoid more than 3 indentation levels inside a
  function — extract a helper instead.
- Internal/helper functions are prefixed with `_` (still snake_case), e.g.
  `_eval_conditions`.

## Comments

- Document public functions with a block comment: purpose and params, brief
  and factual. Grey Hack enforces a per-file character limit — no verbose
  prose, no inline "example output" transcripts (those belong in the
  README/demo file, not `cid.src`).
- Don't comment what the code already says.
- Single-line comments are fine but rare — only when something genuinely
  needs clarifying, and explain the *why*, not the *what*.

## Workflow

- Build incrementally: smallest useful piece first.
- Grey Hack's GreyScript runtime has quirks of its own, and there's no local
  interpreter to test against. Every new addition must be tested live
  in-game before the next feature builds on top of it — keep PRs scoped to
  one testable feature at a time.

## Feature decisions (not yet implemented)

1. `CID.connect(...)` returns a fresh, independent instance every call. The
   old BinDB shared one global instance, so a second `connect()` silently
   clobbered the first. Independent instances let a project split data
   across multiple files, e.g. `loot.db`, `config.db`, `libs.db`.
2. A Drizzle-inspired fluent query builder replaces one-shot helpers like the
   old `fetchBy(table, key, value)`, so filtering on more than one condition
   is possible.
3. Queries always return full row objects — no column projection/select-list
   like real SQL.
4. `where()` must support composable `and`/`or` conditions (Drizzle-style),
   not just a flat left-to-right chain.
5. `join` is explicitly deferred — nice to have, not near-term. Until then,
   relate tables by storing IDs and issuing multiple queries.
6. `update`/`delete` operate on row IDs (via `uuid.src`), not array index —
   the old implementation indexed by array position, which shifts on delete.
7. Queries support `limit` and `offset`.
8. Writes stay in RAM until `.write()` is called explicitly; a crash before
   that loses unsaved changes since last write. Accepted trade-off, not a bug
   to fix.

## Status

No implementation yet. Only `README.md`, `LICENSE.md`, and this file exist at
the repo root.
