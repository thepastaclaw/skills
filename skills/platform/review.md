# Dash Platform — Review Skill

How to weigh findings on `dashpay/platform`. The architectural rules themselves
are defined in the core review context that precedes this file; this file sets
severity, scope, and where to look hardest. It does not restate the rules.

## Severity Policy

**Blocking** — the invariants named in the core context, violated:
- Dependency direction reversed (see the core context and the book's "Where
  code goes"; `dash-sdk` must not depend on `drive-abci`)
- `server` feature of Drive enabled from a client crate
- A consensus-critical method with no `platform_version` dispatch, or an
  existing `v{N}` implementation edited in place
- Consensus error variants reordered, or a code reused/shifted
- `StateTransition` (or any platform-serialized enum) variants inserted
  anywhere but the end
- A state-transition validation stage skipped or reordered
- State reads leaking into `into_high_level_drive_operations`, or writes
  happening outside action application
- Cost/fee divergence between estimation and execution paths, or between
  `prepare_proposal` and `process_proposal`
- An ABCI handler that rejects, stalls on or diverges for a sequence an
  honest Tenderdash network produces (round change, a locked block
  re-proposed in a later round, late votes, restart/replay)

**Blocking — non-consensus correctness.** Most Platform review traffic is
wallet, SDK and FFI code; consensus is not the only blocking class:
- Loss, corruption or misreport of user funds or persisted wallet state
  (including "unknown" shown or stored as zero or as absent)
- A double-send or re-send of a payment, or a spend built on coins the
  network does not have
- A race that changes persisted state (lost update, stale overwrite,
  teardown before a guard)
- The feature's primary path broken (a launch read that throws, an
  operation that can never complete)
- Build, CI or release broken on the base branch by this change

**Blocking — layer ownership** (the book's "Where code goes" and "FFI and
mobile SDKs hold no business logic"):
- New business logic in an FFI crate (`rs-sdk-ffi`, `rs-platform-wallet-ffi`,
  `key-wallet-ffi`), a WASM binding, or the Swift/Kotlin SDKs: the wrapper
  decides something (an index, a path, which key, how many, whether to
  retry, how to rebuild or replay an operation) or reconstructs state or
  identity the Rust side owns
- Validation duplicated in a binding or SDK instead of living once in its
  owner
- A shipped versioned generation edited in place when the edit can change
  behaviour for a protocol version that selects it (the book allows inert
  in-place edits; the PR must show the edit is inert)

**Suggestion — upstream ownership.** A change that mainly works around a
defect or missing capability in another Dash-owned project (rust-dashcore's
key-wallet or dash-spv, tenderdash, grovedb): one finding naming the owning
project and the minimal upstream fix. A relevant linked upstream issue or PR
satisfies it; do not repeat it once linked. It becomes blocking when the
workaround itself breaks a rule above (for example, it lives in the FFI
layer) or is wrong.

**Suggestion** — correctness or maintainability issues outside the classes
above: missing error context, avoidable allocations on hot paths, borderline
placement calls, missing test coverage for a non-consensus branch.

**Nitpick** — naming, comment wording, formatting `cargo fmt` would not catch.

Do not flag what tooling already enforces: `cargo fmt` and `cargo clippy`
(warnings are errors) run in CI.

## Wallet State and Lifecycle Checklist (rs-platform-wallet, Swift/Kotlin SDKs)

Most wallet bugs that reach maintainers only show up when you list state
sequences. For each persisted state and each event the PR touches:

- List every state × transition the PR names, plus re-entry, retry, offline,
  locked device / keychain unavailable, first launch, background launch,
  restore, wipe, and removal while an operation is in flight.
- Trace nil / unknown / unreadable into every aggregate and decision: unknown
  must not become zero, absent, or "no wallet"; a read failure is not absence.
- Check early-return order against teardown and guards (teardown before a
  guard, a guard released before the work it protects).
- Check snapshot-then-store paths for lost updates and stale overwrites (a
  newer state overwritten by a re-emit of an older one).
- For a new launch or background entry point, list the existing reads that
  now run in that context and check each.
- For a gate that blocks the user (a hold, a migration card), list every
  state in which it can never release.

## Error Handling

- Use the `?` operator with proper error types
- No `unwrap()` in production code
- `.expect()` only with a documented justification explaining why the value is
  guaranteed to exist. "It can't be None here" without a stated invariant is not
  a justification.

## Cost Tracking

- Operations must return `CostResult` and propagate costs through the whole call
  chain, including early-return paths
- Cost miscalculations are consensus-critical — every node must compute the same
  cost for the same operation
- New CPU-heavy work must push a corresponding cost operation, not just run

## Proof Correctness

Changes to proof generation or verification need extra scrutiny:
- Proofs must be deterministic across all nodes
- Verification must match generation exactly
- Any divergence breaks consensus

## Test Expectations

- Consensus-affecting changes must have tests; a new versioned method needs
  coverage of the new version *and* evidence the old version still behaves as
  before
- Unit tests in `#[cfg(test)]` modules; integration tests in `tests/`
- Strategy tests for multi-block scenarios

## Conventions to Check

The book's `book/src/contributing/coding-conventions.md` on the reviewed
revision is the rulebook; read the sections the diff touches. Recurring misses:
- Observability uses `tracing` spans with `scope.phase` names, not hand-rolled
  timers or cryptic short labels
- Reuse the existing helper for the pattern (for example `wasm-dpp2`'s
  `try_from_options*` option parsing) instead of hand-rolling a copy
- A shipped versioned generation is frozen; new behaviour goes in a new `vN`
  and its numbers in the version tables
- When a fix touches one instance of a shared pattern (key selection, a proof
  gate, a `FromProof` impl), check every other instance

## Consensus-Critical Code Areas

Extra scrutiny required for:
- `packages/rs-drive-abci/` — block execution and state transition processing
- `packages/rs-drive/` — state storage and proof generation
- `packages/rs-dpp/` — data model validation and serialization
- `packages/rs-platform-version/` — version dispatch tables
- `packages/data-contracts/` and the per-contract crates — system contract
  definitions and migrations

## Known Patterns / Accepted Exceptions

(Populated by feedback loop as false positives are identified.)

## Things NOT to Flag

(Populated by feedback loop as recurring false positives emerge.)
