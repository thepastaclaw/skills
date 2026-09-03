# Dashcore Library — Review Skill

## Review Posture

- Keep findings tightly scoped to the PR. Do not request unrelated modernization or broad Buffer/Uint8Array rewrites unless the PR explicitly claims that scope.
- This is a protocol/serialization library: small byte-handling changes can break downstream consumers. Prefer evidence from concrete fixtures, round trips, and existing behavior.
- Distinguish input widening from output changes. Widening accepted inputs can be non-breaking; changing returned types, instance inheritance, or method availability is often breaking.

## Expected Gates

Primary gates from `package.json`:

```bash
npm test
npm run test:types
npm run test:node
npm run test:browser
npm run lint
npm run build
```

If full `npm test` fails on known baseline `GovObject`/`Proposal` `Invalid Timespan` failures, reviewers should still require focused tests for touched code and call out the baseline explicitly instead of treating it as a PR regression.

## Byte / Serialization Rules

- Preserve byte order exactly. Check little-endian vs big-endian reads/writes, hash reversal, and hex/base64 conversions.
- Do not replace `Buffer` APIs with plain `Uint8Array` unless all methods used are available or inputs are normalized to `Buffer` before Buffer-only methods (`readUInt32LE`, `writeUInt32LE`, etc.).
- `Buffer` extends `Uint8Array` in Node. `value instanceof Uint8Array` accepts both; `Buffer.isBuffer(value)` distinguishes Buffer from plain Uint8Array.
- `Buffer.from(uint8array)` copies bytes. `Buffer.from(arrayBuffer, offset, length)` can share memory; be intentional about copy vs view semantics.
- Do not change return types from Buffer to Uint8Array unless the PR explicitly declares a breaking API change and updates types/docs/tests.
- When adding helper predicates, keep names honest: `isBuffer` should mean Buffer if callers need strict Buffer behavior; use names like `isBytes` for Buffer-or-Uint8Array semantics.

## Constructor / API Detection Rules

- Constructors are intentionally permissive, but detection order matters. Byte-like inputs should not accidentally fall through to generic object handling.
- Static methods named `fromBuffer` historically imply Buffer input. If a PR widens them, verify docs/types/tests and compatibility.
- Existing Buffer callers must keep working unchanged when a PR claims backward compatibility.

## Browser Compatibility Rules

- Browser-targeted changes must avoid introducing new Node-only globals at module load time unless already required by that module.
- Karma/browser tests or build output matter for changes that claim browser compatibility.
- Do not assume all downstream bundlers polyfill Buffer, process, or Node crypto.

## Tests Expected for Byte Handling PRs

- Include explicit plain `Uint8Array` tests using `new Uint8Array(...)`, not just `Buffer.from(...)`.
- Compare behavior against existing Buffer input path on real fixtures when possible.
- Verify typed reads/writes and round trips where boundary normalization is involved.
- Keep existing Buffer-path tests passing.
- Type declaration changes need `test-d` / `npm run test:types` coverage when public types change.

## Common High-Risk Areas

- `lib/encoding/bufferreader.js` / `bufferwriter.js` typed reads/writes and slice position advancement.
- `lib/block/blockheader.js` parsing and hash serialization.
- `lib/transaction/**` varints, payloads, scripts, sighash, and serialization length accounting.
- `lib/deterministicmnlist/**` binary formats and quorum hashes/signatures.
- `lib/instantlock/**` / `lib/chainlock/**` signature payload construction.

## Things Not To Flag

- Continued internal Buffer use when the PR only widens input acceptance.
- Existing module-scope Buffer/runtime dependency unless the PR explicitly claims Buffer-free library loading.
- Existing baseline test failures unrelated to touched code, provided focused touched-code tests pass and the failure is documented.
