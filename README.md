# CID: Central Intelligence Database

CID (Central Intelligence Database) is a GreyScript database library for the game Grey Hack.

It started as a from-scratch rewrite of [BinDB](https://github.com/tuonux/gh-bindb) by [tuonux](mailto:tuonux0@gmail.com) — full credit to the original project for the core idea of a password-protected, binary-compiled database for GreyScript. CID takes that idea in a different direction with a more SQL-like query interface.

## Why this exists

CID is being built as the main database layer for Smoke, a spy-themed all-in-one hacking tool for Grey Hack. That said, CID is developed as its own standalone, plug-and-play library: it has no dependency on Smoke and can be dropped into any GreyScript project.

## Status

This repository is being rewritten from the ground up, one small testable piece at a time. The previous BinDB-derived implementation is preserved as-is on the [`original_base`](https://github.com/FrancoCab10/CID/tree/original_base) branch for reference.

## How to use it

At the top of your code, import `uuid.src` and `cid.src` with the `import_code` method.

```
import_code("/absolute/path/of/uuid.src")
import_code("/absolute/path/of/cid.src")
```

## Available methods

### CID.connect(dbName, dbPassword, dbTables, dbPath)

Creates and returns a new, independent database instance. Every call returns its own instance, so a project can split its data across multiple databases (e.g. `hosts.db`, `config.db`, `sessions.db`) at the same time. `dbPath` defaults to `/root` when omitted.

If a `.db` file already exists at the resolved path, its data is loaded back automatically (see `CID.read()`) — otherwise the instance starts with empty tables.

Example Usage:

```
hostsDb = CID.connect("hosts", "mypassword", ["hosts"], "/home/<user>")
configDb = CID.connect("config", "otherpassword", ["settings"], "/home/<user>")
```

### CID.insert(table)

Starts an insert builder for a table. Chain `.values(data)` to set the row, then `.execute()` to run it and get back the inserted row. A fresh id is generated for the row, overwriting any `id` present in `data`, so rows can later be looked up, updated, or deleted by id. The returned row is a copy — modifying it doesn't change the stored table; use `update()` for that.

On failure, `.execute()` returns a plain string with the error message instead of the row — check `typeof(result) == "string"` to tell them apart. CID never prints errors itself; that's left to your code.

Example Usage:

```
host = hostsDb.insert("hosts").values({"public_ip": "200.43.192.35", "local_ip": "192.168.0.10", "root_pw": "hunter2"}).execute()
print(host.id)
```

### CID.update(table)

Starts an update builder for a table. Chain `.set(data)` to choose the fields to change, `.where(condition)` to scope which rows (see the `where(condition)` section under `CID.query` for the condition helpers — same ones), then `.execute()` to run it. Skipping `.where()` updates every row.

This is a PATCH, not a replace: `.set(data)` merges `data` into each matching row, so only the fields you pass change — everything else on the row is left as-is. `id` is never patched, even if present in `data` — it's fixed at insert (see `CID.insert`), since this engine has no foreign-key protection and a changed `id` would silently orphan anything referencing it. Returns a list of the patched rows (copies, same as `query()`), or an empty list if nothing matched.

On failure (unknown table, `data` isn't a map) returns a plain string with the error message instead — check `typeof(result) == "string"` to tell them apart.

Example Usage:

```
hostsDb.update("hosts").set({"root_pw": "newpassword"}).where(CID.eq("public_ip", "200.43.192.35")).execute()
```

### CID.delete(table)

Starts a delete builder for a table. Chain `.where(condition)` to scope which rows (same condition helpers as `CID.query`/`CID.update`), then `.execute()` to run it. Skipping `.where()` deletes every row.

Returns a list of the deleted rows (copies), or an empty list if nothing matched. On failure (unknown table) returns a plain string with the error message instead — check `typeof(result) == "string"` to tell them apart.

Example Usage:

```
removed = hostsDb.delete("hosts").where(CID.eq("public_ip", "200.43.192.35")).execute()
print(removed.len)
```

### CID.query(table)

Starts a query builder for a table. Chain `.where(condition)` to filter, `.order_by(field, direction)` to sort, `.offset(n)`/`.limit(n)` to page, then `.execute()` to run it and get back the matching rows. Skipping any of them leaves that step out entirely — no `.where()` matches every row, no `.order_by()` keeps insertion order. Rows come back as copies (fresh list, fresh row maps) so nothing in the result shares state with the stored table.

Use `.first()` instead of `.execute()` to get back a single row instead of a list — same pipeline, but returns just the first matching row, or `null` if nothing matched, so callers don't need to check the list's length before indexing into it. On failure it returns the error string unchanged, same as `.execute()`.

Pipeline order is fixed: filter, then sort, then offset, then limit — same as SQL's `WHERE` → `ORDER BY` → `OFFSET` → `LIMIT`.

- `.order_by(field, direction)` — `direction` is `"asc"` (default) or `"desc"`
- `.offset(n)` — skip the first `n` matching rows
- `.limit(n)` — return at most `n` rows

Example Usage:

```
for host in hostsDb.query("hosts").execute()
        print(host.public_ip)
end for
```

#### where(condition)

Build the condition with the comparison and combinator helpers below, then pass the result to `.where()`. There's no varargs in GreyScript, so `every`/`some` take a list rather than multiple arguments — that's the one place this differs from Drizzle's own syntax.

- `CID.eq(field, value)`, `CID.ne(field, value)`, `CID.gt(field, value)`, `CID.gte(field, value)`, `CID.lt(field, value)`, `CID.lte(field, value)` — leaf comparisons
- `CID.like(field, pattern)` — case-insensitive match, `%` as wildcard (`"%foo"`, `"foo%"`, `"%foo%"`, or `"foo"` for exact). A literal `%` at either edge is escaped by doubling it: `"20%%"` matches the value `"20%"` exactly, `"%%foo"` matches a value starting with `"%foo"`. Only edge `%`s need escaping — one in the middle of the pattern (e.g. `"%50%off%"`) is already treated as a literal character, since only the first/last character of the pattern is ever read as a wildcard.
- `CID.every([condition, ...])`, `CID.some([condition, ...])` — combine any number of conditions, and nest them arbitrarily deep

Example Usage:

```
hosts = hostsDb.query("hosts").where(
        CID.every([
                CID.eq("public_ip", "200.43.192.35"),
                CID.some([
                        CID.eq("local_ip", "192.168.0.5"),
                        CID.eq("local_ip", "192.168.0.10")
                ])
        ])
).execute()
```

#### order_by(field, direction) / offset(n) / limit(n)

Example Usage — top 3 hosts by public IP descending, skipping the first one:

```
hosts = hostsDb.query("hosts").order_by("public_ip", "desc").offset(1).limit(3).execute()
```

### CID.write()

Compiles the in-memory tables into the binary database at the configured path, replacing any existing file there. The compiled binary only unlocks its data when launched with the correct password; anyone else who tries to run it directly just sees an info message.

`insert`/`update`/`delete` never call this for you — coming from a traditional DB, that's the one counter-intuitive part: `insert()` alone doesn't persist anything, it just updates the in-memory copy. Call `.write()` yourself when you're ready to flush. It's kept explicit because `write()` recompiles the whole database to disk each time; auto-writing on every insert would turn a batch of inserts into that many full recompiles.

Returns `1` on success. On failure (staging file couldn't be created, compilation failed, ...) returns a plain string with the error message instead — check `typeof(result) == "string"` to tell them apart.

Example Usage:

```
hostsDb.insert("hosts").values({"public_ip": "200.43.192.35", "local_ip": "192.168.0.10", "root_pw": "hunter2"}).execute()
hostsDb.write()
```

### CID.read()

Reloads `self.tables` from the compiled binary at the configured path, if one exists there — overwriting the in-memory tables with whatever was last written, discarding any unwritten changes. `CID.connect()` already calls this once for you, so you don't need it on a fresh connection; call it again yourself later to pick up changes another script wrote to the same file while your database instance was already open.

If no file exists at the path yet, this just resets `self.tables` to empty for each declared table — that's also what makes a fresh `connect()` start empty.

Example Usage:

```
hostsDb.read()
```
