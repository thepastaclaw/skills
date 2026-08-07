# GroveDB — On-Demand Reference

Subsystem detail loaded only when the diff touches the matching paths. The
always-loaded architectural context is in `review-core.md`; nothing here
repeats it.

Each `##` section below opens with an HTML-comment path marker listing its
globs. The orchestrator matches those globs against the lane's changed files and
injects only the matching sections. Sections marked `(manual)` are never loaded
automatically.

## GroveDB Core — File Map
<!-- paths: grovedb/** -->

- `grovedb/src/lib.rs` — public API, doc examples, module structure,
  `start_visualizer`
- `grovedb/src/operations/insert/mod.rs` — insert with element validation
- `grovedb/src/operations/delete/mod.rs` — delete, including `delete_up_tree`
  for cascading removal
- `grovedb/src/operations/get/` — get operations and the `MAX_REFERENCE_HOPS`
  constant; public `GroveDb::follow_reference`
- `grovedb/src/operations/proof/generate.rs` — multi-tree proof generation
- `grovedb/src/operations/proof/` — proof verification (`verify` feature)
- `grovedb/src/reference_path.rs` — internal `follow_reference` /
  `follow_reference_once` with cycle detection
- `grovedb/src/batch/mod.rs` — batch operation machinery
- `grovedb/src/merk_cache.rs` — MerkCache for keeping subtrees open during
  batch ops
- `grovedb/src/error.rs` — the `Error` enum
- `grovedb/src/debugger.rs` — visualizer backend
- Specialized tree operations: `grovedb/src/operations/commitment_tree.rs`,
  `mmr_tree.rs`, `bulk_append_tree.rs`, `dense_tree.rs`

## Merk — File Map
<!-- paths: merk/** -->

- `merk/src/tree/mod.rs` — `TreeNodeInner`, AVL core
- `merk/src/tree/link.rs` — `Link` enum (Reference/Modified/Uncommitted/Loaded)
- `merk/src/tree/walk/mod.rs` — Walker pattern for lazy loading
- `merk/src/tree/ops.rs` — put, delete, batch apply
- `merk/src/tree/commit.rs` — commit logic and hash recomputation
- `merk/src/tree/hash.rs` — Blake3 hashing: `node_hash`, `kv_hash`, `value_hash`
- `merk/src/tree/kv.rs` — `KV` with value-defined cost types
- `merk/src/tree/tree_feature_type.rs` — `TreeFeatureType`, `AggregateData`
- `merk/src/proofs/mod.rs`, `tree.rs`, `query/` — proof ops, execution, and
  encoding/decoding
- `merk/src/proofs/query/verify.rs` — proof verification
- `merk/src/proofs/chunk/` — chunked restoration for replication/sync
- `merk/src/owner.rs` — reference-counting wrapper

## Storage & Supporting Crates — File Map
<!-- paths: storage/**, costs/**, path/**, grovedb-element/**, grovedb-query/**, grovedb-version/**, grovedb-epoch-based-storage-flags/**, grovedbg-types/**, visualize/** -->

- `storage/src/lib.rs` — `Storage`, `StorageContext`, `StorageBatch` traits
- `storage/src/rocksdb_storage/storage.rs` — RocksDB impl, column family names,
  Blake3 prefix generation
- `storage/src/rocksdb_storage/storage_context.rs` — prefixed contexts for
  subtree isolation
- `storage/src/worst_case_costs.rs` — worst-case cost estimation
- `costs/` — `OperationCost`, `StorageCost`, cost macros
- `grovedb-element/` — `Element` enum, serialization, `ReferencePathType`
- `grovedb-query/` — `Query`, `QueryItem`, `PathQuery`
- `grovedb-version/` — `GroveVersion` and the feature-gating macros
- `path/` — `SubtreePath`, `SubtreePathBuilder`
- `grovedb-epoch-based-storage-flags/` — epoch-aware storage flag utilities
- `grovedbg-types/`, `visualize/` — types and tooling for the grovedbg debugger

## Testing
<!-- paths: **/tests/**, **/tests.rs, grovedb/src/tests/**, tutorials/** -->

What a change is expected to be tested for:

1. **Proof verification** — every state-modifying operation should be testable
   end to end: insert data, generate a proof, verify it independently against
   the root hash alone
2. **Cost accuracy** — tests must verify computed costs against actual storage
   operations, including the estimated-cost path. Cost divergence is a consensus
   failure in production, so an untested cost change is an untested consensus
   change.
3. **Reference integrity** — references resolve correctly, cycles are detected,
   hop limits are enforced, dangling references produce the right error
4. **Version compatibility** — tests run against multiple `GroveVersion`
   variants so old-version behavior is pinned, not just the new path
5. **Batch atomicity** — all-or-nothing semantics; partial batch failures must
   roll back completely
6. **Edge cases** — empty trees, maximum-depth paths, boundary key values,
   concurrent transaction conflicts

## Development, Performance & Debugging
<!-- paths: (manual) -->

Not loaded automatically. Pull this in when the review question is specifically
about adding a feature end to end, tuning performance, or debugging tooling.

### Adding a Feature
1. Gate on `GroveVersion` via `check_grovedb_v0_with_cost!` or a sibling macro
2. Implement cost calculation alongside the functionality — costs are
   consensus-critical, not optional
3. Make sure proof generation covers the new feature
4. Add batch operation support if applicable
5. Test edge cases and cost accuracy, not just the happy path

### Performance
1. Use batch operations for multiple changes — avoids per-operation hash
   propagation
2. Leverage `MerkCache` for repeated access to the same subtrees inside a
   transaction
3. Minimize tree opens by using persistent storage contexts
4. Consider cost limits on expensive operations to prevent resource exhaustion
5. Use lazy loading — don't force-load whole subtrees for a few keys
6. Estimated costs (feature-gated) allow pre-flight checks without executing

### Debugging
1. **Visualizer**: `db.start_visualizer(addr)` launches a web tree inspector
   (backend in `grovedb/src/debugger.rs`)
2. **Cost analysis**: log `OperationCost` values to find expensive operations
3. **Proof verification**: test proofs independently to isolate whether a bug is
   in generation or verification
4. **Reference tracing**: follow reference chains manually; check for cycles
5. **Version checks**: confirm the correct `GroveVersion` is threaded through
   every call site

## Workspace & Build
<!-- paths: (manual) -->

Not loaded automatically.

Workspace members (root `Cargo.toml`): `costs`, `grovedb`, `merk`, `storage`,
`visualize`, `path`, `grovedbg-types`, `grovedb-version`,
`grovedb-epoch-based-storage-flags`, `grovedb-element`, `grovedb-commitment-tree`,
`grovedb-merkle-mountain-range`, `grovedb-bulk-append-tree`,
`grovedb-dense-fixed-sized-merkle-tree`, `grovedb-query`. Per-crate
documentation lives under `docs/crates/`; `adr/` holds architecture decision
records.

```sh
cargo build
cargo test
cargo test -p grovedb                        # Core crate only
cargo test -p merk                           # Merk crate only
cargo test -p storage                        # Storage crate only
cargo test test_name                         # Specific test
cargo test --features full,estimated_costs   # With all features
cargo clippy -- -D warnings
cargo bench
```
