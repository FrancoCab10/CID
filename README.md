# CID: Central Intelligence Database

CID (Central Intelligence Database) is a GreyScript database library for the game Grey Hack.

It started as a from-scratch rewrite of [BinDB](https://github.com/tuonux/gh-bindb) by [tuonux](mailto:tuonux0@gmail.com) — full credit to the original project for the core idea of a password-protected, binary-compiled database for GreyScript. CID takes that idea in a different direction with a more SQL-like query interface.

## Why this exists

CID is being built as the main database layer for Smoke, a spy-themed all-in-one hacking tool for Grey Hack. That said, CID is developed as its own standalone, plug-and-play library: it has no dependency on Smoke and can be dropped into any GreyScript project.

## Status

This repository is being rewritten from the ground up. The previous BinDB-derived implementation is preserved as-is on the [`original_base`](https://github.com/FrancoCab10/CID/tree/original_base) branch for reference. The new implementation, API, and docs will land here incrementally.
