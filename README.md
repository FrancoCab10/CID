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

Creates and returns a new, independent database instance. Every call returns its own instance, so a project can split its data across multiple databases (e.g. `loot.db`, `config.db`, `libs.db`) at the same time.

Example Usage:

```
lootDb = CID.connect("loot", "mypassword", ["items", "credits"], "/home/<user>")
configDb = CID.connect("config", "otherpassword", ["settings"], "/home/<user>")
```

### CID.insert(table)

Starts an insert builder for a table. Chain `.values(data)` to set the row, then `.execute()` to run it and get back the inserted row. A fresh id is generated for the row, overwriting any `id` present in `data`, so rows can later be looked up, updated, or deleted by id.

On failure, `.execute()` returns a plain string with the error message instead of the row — check `typeof(result) == "string"` to tell them apart. CID never prints errors itself; that's left to your code.

Example Usage:

```
item = lootDb.insert("items").values({"name": "lockpick", "quantity": 3}).execute()
print(item.id)
```
