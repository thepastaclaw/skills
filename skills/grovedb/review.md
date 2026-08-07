# GroveDB — Review Skill

How to weigh findings on `dashpay/grovedb`. The architectural rules themselves
are defined in the core review context that precedes this file; this file sets
severity, scope, and where to look hardest. It does not restate the rules.

## Severity Policy

**Blocking** — the invariants named in the core context, violated:
- Proof generation changed without the matching verification change (or vice
  versa), or proof output made non-deterministic
- A hash, encoding, or root-propagation change that is not `GroveVersion`-gated
- Merk invariants broken: balance factor outside `{-1, 0, 1}`, a rotation that
  loses sorted order, or an aggregate that fails to propagate on rebalance
- Link state machine skipped — reading a hash from a `Modified` link, or
  committing without passing through `Uncommitted`
- Cost divergence: uncharged work, cost dropped by a bare `?` where a
  `cost_return_on_error!` macro belongs, or estimated cost that does not match
  executed cost
- Reference resolution without hop limit or cycle detection, or hops that are
  not charged
- A batch that can leave state partially applied
- Storage prefix generation made non-deterministic, or prefix isolation broken

**Suggestion** — correctness or maintainability issues off the consensus path:
missing error context, an over-broad error variant, avoidable tree opens,
missing coverage for a non-consensus branch.

**Nitpick** — naming, comment wording, formatting `cargo fmt` would not catch.

Do not flag what tooling already enforces: `cargo fmt` and
`cargo clippy -- -D warnings` run in CI.

## Error Handling

- Use `cost_return_on_error!` (or its no-add/into variants) for cost-aware
  propagation; a bare `?` on a cost-returning call silently drops accumulated
  cost
- Proper `Error` types throughout — no `unwrap()` in production code
- `.expect()` only with a documented justification naming the invariant that
  guarantees the value

## Test Expectations

- Proof verification tests for any proof-related change
- Cost accuracy tests verifying exact cost calculations, including the
  estimated-cost path
- Reference integrity tests: cycles, hop limits, missing targets
- Batch atomicity tests: partial failure and rollback
- AVL balance tests after insertions and deletions
- Multi-`GroveVersion` tests when behavior is version-gated, so the old path
  stays pinned

## Security-Critical Areas

Extra scrutiny required for:
- **Proof generation/verification** (`grovedb/src/operations/proof/`,
  `merk/src/proofs/`) — any change can break consensus
- **Reference resolution** (`grovedb/src/reference_path.rs`,
  `grovedb/src/operations/get/`) — cycle and hop-limit vulnerabilities
- **Batch operations** (`grovedb/src/batch/`, `grovedb/src/merk_cache.rs`) —
  atomicity and rollback correctness
- **Storage prefix generation** (`storage/src/rocksdb_storage/`) — subtree
  isolation integrity
- **Merk rotations and hash updates** (`merk/src/tree/`) — tree invariant and
  aggregate preservation

## Known Patterns / Accepted Exceptions

(Populated by feedback loop as false positives are identified.)

## Things NOT to Flag

(Populated by feedback loop as recurring false positives emerge.)
