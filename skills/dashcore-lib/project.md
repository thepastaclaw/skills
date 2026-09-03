# Dashcore Library — Project Skill

## Project Purpose

`dashpay/dashcore-lib` (`@dashevo/dashcore-lib`) is the JavaScript Dash protocol/library package used for addresses, keys, scripts, transactions, blocks, block headers, Bloom filters, governance objects, InstantLocks/ChainLocks, deterministic masternode lists, and serialization utilities.

It targets both Node.js and browser bundles. Public APIs historically use Node `Buffer` heavily, while downstream browser-targeted Platform packages increasingly prefer plain `Uint8Array` surfaces.

## Tech Stack

- CommonJS JavaScript library.
- Type declarations in `index.d.ts` / `typings/` validated by `tsd`.
- Node tests: Mocha + Chai.
- Browser tests: Karma + webpack.
- Lint: ESLint Airbnb base + Prettier.
- Build: webpack browser bundle.

## Repository Layout

- `lib/encoding/` — BufferReader/BufferWriter, base58/base58check, varint.
- `lib/block/` — Block, BlockHeader, MerkleBlock, PartialMerkleTree.
- `lib/transaction/` — transaction model, sighash, outputs, payloads.
- `lib/crypto/` — BN, ECDSA, hash, point, signatures, BLS wrapper.
- `lib/deterministicmnlist/` — SML/SMLDiff/quorum structures.
- `lib/instantlock/`, `lib/chainlock/` — lock structures/validation.
- `lib/govobject/` — governance/proposal parsing and validation.
- `lib/util/` — shared byte, hash, JS, precondition, IP helpers.
- `test/` mirrors core library areas; `test-d/` covers TypeScript declarations.

## Architectural Notes

- Return types are often `Buffer`; changing outputs is usually breaking and should be explicit.
- Input widening is acceptable when normalized at module boundaries and existing Buffer callers keep working.
- Browser compatibility matters, but the library still depends on Buffer at runtime in many areas. Do not assume Buffer-free operation unless a PR explicitly targets it.
- Serialization/parsing code is security- and consensus-adjacent: byte order, hash reversal, lengths, version fields, and boundary checks must stay exact.
- Many classes follow permissive constructor patterns (`new Foo(buffer|string|object)` plus `Foo.fromBuffer` / `fromObject`). Keep detection order intentional.

## Known Test Caveat

As of 2026-05-26, local full `npm test` on current `master` can fail in existing `GovObject`/`Proposal` tests with `Invalid Timespan` before browser tests run. Focused Mocha tests and `test:types` can still be meaningful, but reviewers should separate pre-existing baseline failures from PR-introduced regressions.
