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

Creates and returns a new, independent database instance. Every call returns its own instance, so a project can split its data across multiple databases (e.g. `loot.db`, `config.db`, `libs.db`) at the same time. `dbPath` defaults to `/root` when omitted.

Example Usage:

```
lootDb = CID.connect("loot", "mypassword", ["items", "credits"], "/home/<user>")
configDb = CID.connect("config", "otherpassword", ["settings"], "/home/<user>")
```

### CID.insert(table)

Starts an insert builder for a table. Chain `.values(data)` to set the row, then `.execute()` to run it and get back the inserted row. A fresh id is generated for the row, overwriting any `id` present in `data`, so rows can later be looked up, updated, or deleted by id. The returned row is a copy — modifying it doesn't change the stored table; use `update()` for that.

On failure, `.execute()` returns a plain string with the error message instead of the row — check `typeof(result) == "string"` to tell them apart. CID never prints errors itself; that's left to your code.

Example Usage:

```
item = lootDb.insert("items").values({"name": "lockpick", "quantity": 3}).execute()
print(item.id)
```

### CID.query(table)

Starts a query builder for a table. Chain `.where(condition)` to filter, then `.execute()` to run it and get back the matching rows — every row if `.where()` is skipped. Rows come back as copies (fresh list, fresh row maps) so nothing in the result shares state with the stored table. `orderBy`/`limit`/`offset` are still to come.

Example Usage:

```
for item in lootDb.query("items").execute()
        print(item.name)
end for
```

#### where(condition)

Build the condition with the comparison and combinator helpers below, then pass the result to `.where()`. There's no varargs in GreyScript, so `every`/`some` take a list rather than multiple arguments — that's the one place this differs from Drizzle's own syntax.

- `CID.eq(field, value)`, `CID.ne(field, value)`, `CID.gt(field, value)`, `CID.gte(field, value)`, `CID.lt(field, value)`, `CID.lte(field, value)` — leaf comparisons
- `CID.like(field, pattern)` — case-insensitive match, `%` as wildcard (`"%foo"`, `"foo%"`, `"%foo%"`, or `"foo"` for exact). There's no escape for a literal `%` in the pattern — a value like `"20%"` can't be matched exactly through `like` since the trailing `%` is always read as a wildcard. Use `CID.eq` instead when the value itself may contain `%`.
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

### CID.write()

Compiles the in-memory tables into the binary database at the configured path, replacing any existing file there. The compiled binary only unlocks its data when launched with the correct password; anyone else who tries to run it directly just sees an info message.

`insert`/`update`/`delete` never call this for you — coming from a traditional DB, that's the one counter-intuitive part: `insert()` alone doesn't persist anything, it just updates the in-memory copy. Call `.write()` yourself when you're ready to flush. It's kept explicit because `write()` recompiles the whole database to disk each time; auto-writing on every insert would turn a batch of inserts into that many full recompiles.

Returns `1` on success. On failure (staging file couldn't be created, compilation failed, ...) returns a plain string with the error message instead — check `typeof(result) == "string"` to tell them apart.

Example Usage:

```
lootDb.insert("items").values({"name": "lockpick", "quantity": 3}).execute()
lootDb.write()
```
