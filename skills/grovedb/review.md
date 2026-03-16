# GroveDB — Review Skill

## Code Style

- `cargo fmt` enforced
- `cargo clippy` with `-D warnings` — all warnings are errors

## Cryptographic Correctness (Highest Priority)

Cryptographic integrity is paramount in GroveDB:
- Proof generation must be deterministic
- Hash consistency across all operations
- Merkle tree invariants must hold at all times
- Any proof integrity violation breaks Platform consensus

## AVL Tree Invariants

- Balance factor must always be in {-1, 0, 1}
- Rotations must preserve sorted order and update hashes
- Link state transitions must follow the state machine:
  Reference → Loaded → Modified → Uncommitted → Reference

## Cost Tracking

- Cost miscalculations affect Platform consensus — all nodes must
  agree on operation costs
- Every operation must accurately track seek_count,
  storage_loaded_bytes, storage_cost, hash_node_calls
- Use `cost_return_on_error!` macro for error handling that
  preserves accumulated costs

## Reference Resolution

- Must enforce hop limits to prevent infinite chains
- Must prevent reference cycles
- Must handle missing targets gracefully with proper errors
- Reference resolution costs must be tracked accurately

## Error Handling

- Use `cost_return_on_error!` macro for cost-aware error propagation
- Proper `Error` types throughout — no `unwrap()` in production code
- `.expect()` only with documented justification

## Batch Atomicity

- Batch operations must be all-or-nothing
- Partial failures must not corrupt state
- Two-phase processing: validate all operations before applying any
- Rollback must leave the tree in a consistent state

## Storage Layer

- Prefix isolation must be maintained — no cross-subtree data leakage
- Blake3 prefix generation must be deterministic
- Storage type selection must be correct for the data category

## Version Compatibility

- Changes must work across grove versions
- Version dispatch must be correct

## Test Expectations

- Proof verification tests for any proof-related changes
- Cost accuracy tests verifying exact cost calculations
- Reference integrity tests (cycles, hop limits, missing targets)
- Batch atomicity tests (partial failure, rollback)
- AVL balance tests after insertions/deletions

## Known Patterns / Accepted Exceptions

- `cost_return_on_error!` macro usage throughout — this is the
  standard error handling pattern
- Link state machine transitions — complex but correct by design
- Blake3 prefix generation — deterministic subtree isolation
- Version dispatch boilerplate

## Things NOT to Flag

- Proof system complexity — it is inherently complex and the
  complexity is necessary
- Version dispatch boilerplate
- Cost tracking verbosity — every operation must track costs
- `cost_return_on_error!` macro density — this is expected

## Security-Critical Areas

Extra scrutiny required for:
- **Proof generation/verification** — any change can break consensus
- **Reference resolution** — cycle/hop vulnerabilities
- **Batch operations** — atomicity and rollback correctness
- **Storage prefix generation** — subtree isolation integrity
- **Merk rotations and hash updates** — tree invariant preservation
