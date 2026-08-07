# Dash Platform — Review Skill

How to weigh findings on `dashpay/platform`. The architectural rules themselves
are defined in the core review context that precedes this file; this file sets
severity, scope, and where to look hardest. It does not restate the rules.

## Severity Policy

**Blocking** — the invariants named in the core context, violated:
- Dependency direction (DPP → Drive → Drive-ABCI → SDK) reversed
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

**Suggestion** — correctness or maintainability issues that do not put
consensus at risk: missing error context, avoidable allocations on hot paths,
unclear ownership, missing test coverage for a non-consensus branch.

**Nitpick** — naming, comment wording, formatting `cargo fmt` would not catch.

Do not flag what tooling already enforces: `cargo fmt` and `cargo clippy`
(warnings are errors) run in CI.

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
