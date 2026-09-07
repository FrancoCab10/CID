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
- `CID` is the only class in the file. Insert/update/delete/query builders
  are plain maps (`{}`), not separate classes — wire up shared behavior with
  helper functions referenced via `@`, e.g. `builder.values = @CID["_values"]`,
  so logic used by more than one builder (like setting the data payload)
  isn't duplicated per class. Use bracket form (`@CID["_values"]`), not
  `@CID._values` — confirmed live in-game that the dotted form doesn't
  behave the same as a bare `@_values` did, even though both run fine
  through greybel's local Mock interpreter. Mock is not a substitute for
  live testing on anything involving `@`.
- Every function lives under the `CID` namespace, public and internal alike
  — no bare globals, not even helpers. `import_code` dumps everything into
  the caller's global scope, so a bare `_values` or `_where` is exactly the
  kind of short name a consumer's own script might already have.

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
- Internal/helper functions are prefixed with `_` and namespaced under
  `CID` (still snake_case), e.g. `CID._eval_condition`.

## Comments

- `//` only. There's no block comment syntax in this project — `/* */` is a
  greybel-only extension that Grey Hack's in-game code editor doesn't
  understand, so a raw `.src` copy-pasted straight into the game would hit a
  syntax error. Tried that plus a CI step to transpile it; decided it wasn't
  worth the complexity, so `//` it is, everywhere.
- Every function gets its own `//` block directly above it, even near-
  identical ones (`CID.eq`/`CID.ne`/`CID.gt`/...) — never share one comment
  across a cluster of functions. The greybel language server shows this
  comment as hover documentation per-function; a shared block only
  documents the first function in the group, and the rest show up as
  undocumented. Confirmed live in the greybel language server.

  ```
  // short explanation
  //
  // param_name type
  // param_name type
  // Return type
  //
  // some examples if needed
  ```

- Keep it brief and factual — no verbose prose, no inline "example output"
  transcripts (Grey Hack enforces a per-file character limit; full examples
  belong in the README/demo file, not `cid.src`).
- Don't comment what the code already says.
- A single `//` line is fine but rare — only when something genuinely needs
  clarifying, and explain the *why*, not the *what*.

## Error handling

- Library code never calls `print()` on failure — that takes error-message
  control away from the programmer using the library. On failure, `return`
  the error message as a plain string instead; the caller decides whether to
  print it, log it, or handle it some other way.
- Since success values are otherwise maps/lists/numbers, callers can check
  `typeof(result) == "string"` to detect an error.

## Workflow

- Build incrementally: smallest useful piece first.
- `greybel-js`'s local interpreter (`greybel execute <file> --env-type Mock`)
  is trustworthy for core language logic — control flow, maps/lists,
  recursion, string ops — and is the default way to verify a change before
  pushing. Divergence from real Grey Hack is rare and mainly shows up in
  Grey Hack-specific objects (`get_shell`, `host_computer`, `File`, `build`,
  ...) that can shift between game updates — that's what still needs live
  in-game confirmation, not every change.
- One exception hit already: `@CID._values` (dotted) ran fine under Mock but
  didn't behave the same in-game as `@CID["_values"]` (bracket) — logged
  here, not as a reason to distrust `@` in general, just as the one known
  case where Mock and in-game disagreed on something that wasn't a Grey
  Hack-specific object.
- Keep PRs scoped to one testable feature at a time.

## Feature decisions

1. `CID.connect(...)` returns a fresh, independent instance every call, so a
   project can split its data across multiple files, e.g. `loot.db`,
   `config.db`, `libs.db`, at the same time.
2. Every operation — insert, update, delete, and querying — follows the same
   Drizzle-inspired builder shape: `CID.<verb>(table)` returns a builder;
   chain modifiers (`.values()`, `.set()`, `.where()`, ...) and terminate
   with `.execute()` for writes or `.get()`/`.first()` for reads. E.g.
   `CID.insert(table).values(data).execute()` returns the full inserted row.
   This replaces one-shot helpers like the old `fetchBy(table, key, value)`,
   so filtering on more than one condition is possible.
3. Queries always return full row objects — no column projection/select-list
   like real SQL. Same for insert/update: they return the full row, not just
   its id.
4. `where()` supports composable `and`/`or` conditions (Drizzle-style), not
   just a flat left-to-right chain. Implemented as a condition tree: leaf
   comparisons (`CID.eq`, `CID.ne`, `CID.gt`, `CID.gte`, `CID.lt`, `CID.lte`,
   `CID.like`) and combinators (`CID.every`, `CID.some`) are functions namespaced
   under `CID` — not bare globals like Drizzle's own `eq`/`and`/`or` — since
   `import_code` dumps everything into the caller's global scope and short
   names like that are exactly what a consumer's own script is likely to
   already use. `every`/`some` take a list (no varargs in GreyScript) and
   can nest arbitrarily deep. A shared `CID._where`/`CID._eval_condition`
   pair backs `.where()` so update/delete can reuse it once they land.
5. `join` is explicitly deferred — nice to have, not near-term. Until then,
   relate tables by storing IDs and issuing multiple queries.
6. `update`/`delete` operate on row IDs (via `uuid.src`), not array index,
   since array position shifts whenever a row is deleted.
7. Queries support `limit` and `offset`.
8. Writes stay in RAM until `.write()` is called explicitly; a crash before
   that loses unsaved changes since last write. Accepted trade-off, not a bug
   to fix. `insert`/`update`/`delete` never call `write()` themselves either
   — considered and rejected, since `write()` recompiles the whole database
   to disk, so auto-writing on every single insert would turn a batch insert
   into that many full recompiles. Counter-intuitive coming from a
   traditional DB where insert just persists, but the cost of the
   alternative is worse.

## Status

Implemented so far, in `cid.src`: `CID.connect()` (default `db_path` is
`/root`), `CID.insert(table)` (builder: `.values(data).execute()`),
`CID.query(table)` (builder: `.where(condition).execute()`, with
`CID.eq/ne/gt/gte/lt/lte/like/every/some` condition builders), and
`CID.write()`. `uuid.src` is in the repo and provides the global `uuid()`
function used to assign row ids. `orderBy`/`limit`/`offset` and everything
else in the feature decisions above is still pending.

Known gap: `CID.connect()` doesn't yet load an existing `.db` file's data
back into `self.tables` — every connect() starts from empty tables, even if
a database was already written to that path. Reading the compiled binary
back (the old BinDB.read() did this via get_shell.launch() + get_custom_object)
is still to do. Confirmed live: what looked like a `CID.ne`/`CID.every` bug
was actually this — querying a fresh connect() with no inserts always
returns nothing, regardless of what's on disk.

`CID.like`'s `%` wildcard is escaped by doubling: `"20%%"` matches the
value `"20%"` exactly, rather than being read as a wildcard suffix. Chose
doubling over a backslash escape (`\%`) since it was unclear whether
GreyScript string literals would even preserve a bare backslash through
their own escape processing (`\n` is already interpreted as a real newline
in this codebase's generated write() source) — doubling only depends on
the runtime string's characters, not on how the caller's literal was
parsed. Verified live via `greybel execute`.
