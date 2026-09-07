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

Starts a query builder for a table. Chain `.execute()` to run it and get back the table's rows. No filters yet (`where`/`orderBy`/`limit`/`offset` are still to come) — it returns the whole table, as copies (fresh list, fresh row maps) so nothing in the result shares state with the stored table.

Example Usage:

```
for item in lootDb.query("items").execute()
        print(item.name)
end for
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
