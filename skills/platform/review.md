# Dash Platform — Review Skill

## Code Style

- `cargo fmt` enforced
- `cargo clippy` required — warnings are errors
- Rust edition 2021

## Versioning (Consensus-Critical)

Every consensus-critical method must dispatch on `platform_version`.
Missing version dispatch is a **blocking** issue. The
`PlatformVersion` struct controls which code path executes at each
protocol version. New consensus methods must add version match arms.

## Feature Flag Hygiene

- Never enable `server` feature of Drive in client crates
  (dash-sdk, etc.) — this is **blocking**
- Check that feature flags are correctly gated
- DPP has 80+ feature flags; ensure new code respects existing gates

## Dependency Direction

Violations are **blocking**:
- DPP must not import Drive
- Drive must not import Drive-ABCI
- Client crates must not depend on server-only code

## Error Handling

- Use `?` operator with proper error types
- No `unwrap()` in production code
- `.expect()` only with documented justification explaining why the
  value is guaranteed to exist

## State Transition Validation

State transitions must follow the validation pipeline:
1. `check_tx` — initial validation
2. `process_proposal` — proposal-time validation
3. `finalize_block` — final application

Skipping pipeline stages is a consensus violation.

## Cost Tracking

- Operations must return `CostResult`
- Cost miscalculations are consensus-critical — all nodes must agree
  on costs
- Propagate costs correctly through the call chain

## Proof Correctness

Changes to proof generation or verification need extra scrutiny:
- Proof must be deterministic across all nodes
- Verification must match generation exactly
- Any divergence breaks consensus

## Test Expectations

- Unit tests in `#[cfg(test)]` modules
- Integration tests in `tests/` directories
- Strategy tests for complex multi-step scenarios
- Consensus changes must have tests

## System Contracts

Changes to system contracts need:
- Version bumps
- Migration paths for existing data
- Consideration of upgrade ordering

## Known Patterns / Accepted Exceptions

(Populated by feedback loop as false positives are identified.)

## Things NOT to Flag

(Populated by feedback loop as recurring false positives emerge.)

## Consensus-Critical Code Areas

Extra scrutiny required for:
- `drive-abci/` — block execution and state transition processing
- `drive/` — state storage and proof generation
- `dpp/` — data model validation and serialization
- `platform-version/` — version dispatch tables
- System contract definitions and migrations
