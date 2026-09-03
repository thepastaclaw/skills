# rust-dashcore — Review Skill

## Style and Conventions

- Rust code should be idiomatic and explicit about ownership, error handling, feature gates, and public API compatibility.
- Prefer precise tests that assert semantics, not only that code runs.
- Avoid panics/unwraps in library or FFI-facing code unless the invariant is proven and documented.
- Public API changes should be intentional, documented, and reflected across Rust and FFI surfaces when both expose the same concept.

## Common Pitfalls

- Dropping metadata when converting rich wallet/address structures into simpler event/API payloads.
- Emitting duplicate derived addresses from multiple transactions in a block without a documented dedup key.
- Updating Rust `WalletEvent` variants without updating FFI callback payloads or downstream-facing docs.
- Changing return types of public helpers when a compatibility-preserving adapter would work.
- Treating non-ECDSA pools as if they have compressed 33-byte ECDSA public keys.
- Event payloads that carry stale balances because balance refresh happens after emission.
- Tests that only cover external receive pools while internal/change or index-less account types have different behavior.

## What to Flag

- Incorrect wallet state, missing persistence-critical metadata, broken gap-limit behavior, stale balances, duplicate records that violate the documented event contract, unsafe FFI/ABI behavior, and undocumented public API breaks.
- Missing tests for the specific edge case introduced by the PR when the behavior is subtle or persistence-facing.

## What Not to Flag

- Pure source compatibility breaks that are clearly intentional and acceptable for the current development branch, unless the PR claims no breaking changes or leaves an avoidable wider blast radius.
- Cosmetic formatting/style caught by `cargo fmt` / clippy.
