# GroveDB — Project Understanding

## Overview

GroveDB is a hierarchical authenticated data structure database — a "grove"
(tree of trees) built on Merkle AVL trees backed by RocksDB. It provides
efficient secondary index queries, cryptographic proofs for all queries, and
is optimized for blockchain applications like Dash Platform. Every query
result can be independently verified without trusting the data source,
enabling trustless state verification.

GroveDB's distinguishing property is that it manages **multiple Merk trees
arranged hierarchically**: any element in a tree can itself be a subtree root,
creating an arbitrarily deep grove. This enables path-based addressing
(e.g., `["identities", id, "keys"]`) and composite proofs that span multiple
subtrees in a single response.

## Three-Layer Architecture

```
GroveDB Core  →  Merk  →  Storage
```

### Layer 1: GroveDB Core (`grovedb/src/`)

Orchestrates multiple Merk trees into the hierarchical grove structure.
Responsibilities:

- **Path-based operations**: insert, get, delete addressed by `(path, key)`
  pairs where `path` is a vector of byte-string segments
- **Element management**: 8 element types including references and aggregate
  trees (see Element System below)
- **Reference resolution**: follows cross-tree pointers with hop limits and
  cycle detection
- **Batch processing**: atomic multi-subtree operations via two-phase
  validation-then-application
- **Proof generation**: composite proofs spanning multiple subtrees in one
  response
- **Query execution**: `PathQuery` with conditional subquery branches for
  complex traversals across the grove
- **MerkCache**: keeps frequently accessed subtrees open in memory to avoid
  repeated open/close overhead. Uses `UnsafeCell` internally for interior
  mutability with a borrow-flag guard per cached Merk.

### Layer 2: Merk (`merk/src/`)

A Merkle AVL tree implementation — each individual subtree in the grove is a
Merk instance. Key properties:

- **AVL self-balancing**: balance factor always in `{-1, 0, 1}`, enforced
  via single/double rotations → O(log n) guaranteed
- **Intermediary nodes store key-value pairs**: unlike typical Merkle trees,
  every node (not just leaves) holds data
- **Lazy loading via the Link system**: nodes are loaded from storage only
  when accessed (see Merk Internals below)
- **Cost-aware operations**: every operation accumulates an `OperationCost`
  tracking seeks, bytes read/written, and hash calls
- **Chunk-based restoration**: large trees can be restored incrementally
  for replication/sync
- **Predefined costs for specialized nodes**: aggregate tree types (SumTree,
  CountTree, etc.) have known value sizes, enabling cost prediction

### Layer 3: Storage (`storage/src/`)

Abstracts RocksDB with prefixed storage for subtree isolation.

- **RocksDB backend**: uses `OptimisticTransactionDB` for concurrent
  transaction support
- **Blake3 prefix isolation**: each subtree's path segments are hashed with
  Blake3 to produce a 32-byte prefix. All keys for that subtree are stored
  under this prefix, preventing cross-subtree data leakage
- **Four column families** (storage types):
  - `default` — main tree node data (keys, values, hashes, links)
  - `aux` — auxiliary data associated with subtrees
  - `roots` — subtree root node metadata
  - `meta` — database-level metadata
- **Batch operations**: `StorageBatch` accumulates writes and flushes them
  in a single RocksDB write batch to minimize disk I/O
- **Prefixed contexts**: `PrefixedRocksDbTransactionContext` and
  `PrefixedRocksDbImmediateStorageContext` scope all reads/writes to a
  specific subtree prefix

## Element System

8 element types, each serialized and stored as values in Merk nodes:

| Element | Description | Use Case |
|---------|-------------|----------|
| `Item` | Raw key-value byte storage | Basic data storage (identity fields, contract documents) |
| `Reference` | Pointer to another element anywhere in the grove | Cross-tree links, secondary indexes pointing to primary data |
| `Tree` | Subtree root — creates a new Merk instance beneath this node | Hierarchical organization (e.g., an identity's "keys" subtree) |
| `SumItem` | Item that contributes an integer value to its parent's sum | Individual balance entries, vote counts |
| `SumTree` | Subtree that maintains the sum of all descendant `SumItem` values | Aggregate balance tracking (total platform credits) |
| `BigSumTree` | Like `SumTree` but with 256-bit (i128) sums for large values | High-precision aggregate accounting |
| `CountTree` | Subtree that counts the number of elements it contains | Cardinality tracking (number of keys, number of documents) |
| `CountSumTree` | Combined counting and summing — tracks both element count and value sum | Cases needing both cardinality and aggregate value |

Aggregate tree types (`SumTree`, `BigSumTree`, `CountTree`, `CountSumTree`)
store their aggregate value in the Merk node's feature type field, which
propagates up during tree rebalancing. This means aggregate values are always
available at the root without scanning descendants.

## Reference System

7 reference types enable flexible cross-tree pointers:

1. **AbsolutePathReference** — direct full path from root. Simple but
   brittle if tree structure changes. Example: `["identities", id, "balance"]`

2. **UpstreamRootHeightReference** — navigate up to height N from root,
   then follow a relative path. Useful when the target is in a known
   position relative to an ancestor.

3. **UpstreamFromElementHeightReference** — navigate up N levels from the
   current element's position, then follow a path. Relative to the
   reference's own location.

4. **CousinReference** — same tree depth level, different branch. Navigate
   to a sibling of the parent at the same depth. Used for cross-branch
   links at the same logical level.

5. **SiblingReference** — same parent tree, different key. The simplest
   relative reference — points to another element in the same subtree.

6. **UpstreamRootHeightWithParentPathAddition** — complex navigation: go
   up to a root height, append the parent's path, then follow additional
   segments. Used for pattern-based references in structured data.

7. **UtilityReference** — system-level references for internal bookkeeping.

### Reference Resolution

References are resolved by the `follow_reference` function in
`grovedb/src/reference_path.rs`:

- **Hop limit**: `MAX_REFERENCE_HOPS` (currently defined in
  `operations/get`) prevents infinite chains
- **Cycle detection**: a `HashSet` of visited qualified paths detects loops;
  returns `Error::CyclicReference` if a path is visited twice
- **Chained references**: if a reference points to another reference, it
  continues following until it reaches a non-reference element or hits the
  hop limit (`Error::ReferenceLimit`)
- `follow_reference_once` variant stops at the first resolved element
  without following further reference chains

## Cost Tracking

Every operation accumulates an `OperationCost` struct (from the `costs` crate):

```rust
OperationCost {
    seek_count: u16,             // Number of RocksDB seeks (disk I/O)
    storage_loaded_bytes: u32,   // Bytes read from storage
    storage_cost: StorageCost {  // Write cost breakdown
        added_bytes: u32,        //   New bytes written
        replaced_bytes: u32,     //   Bytes overwritten
        removed_bytes: StorageRemovedBytes,  // Bytes freed
    },
    hash_node_calls: u32,        // Blake3/hash operations performed
}
```

**Why costs matter**: Dash Platform runs GroveDB on every masternode. Consensus
requires all nodes to compute **identical costs** for the same operation — any
divergence means a consensus failure and chain halt. Therefore:

- Cost calculations are part of the consensus-critical path
- Tests verify cost accuracy against actual operations
- The `cost_return_on_error!` macro ensures costs accumulate correctly even
  on early returns
- Estimated costs (via `estimated_costs` feature) allow pre-computation
  without executing operations

### Cost Macros

```rust
// Accumulate costs even when returning early on error
cost_return_on_error!(&mut cost, some_operation());

// Same but doesn't add to cost on error path
cost_return_on_error_no_add!(&mut cost, some_operation());

// Into conversion variant
cost_return_on_error_into_no_add!(cost, some_operation());
```

## Proof System

GroveDB generates **composite proofs** that span multiple subtrees:

### Generation (`grovedb/src/operations/proof/generate.rs`)

- **Layer-by-layer**: starts at the root tree and descends to target
  subtrees, generating a Merk proof at each layer
- **Depth-first traversal**: collects relevant nodes along the path from
  root to each queried subtree
- **Minimal proof size**: excludes unnecessary sibling data — only includes
  nodes on the path to queried keys plus enough to verify the Merkle root
- **Absence proofs**: can prove that a key does NOT exist by including the
  neighboring nodes that would bracket the missing key

### Verification (`proof/` with `verify` feature)

- **Stack-based virtual machine**: proof verification uses a stack-based
  approach where operations (push node, hash combine, etc.) are executed
  sequentially
- **Independent verification**: proofs are self-contained — a verifier needs
  only the proof bytes and the expected root hash, not the database
- **Serialization**: proofs use a compact encoding (`merk/src/proofs/encoding.rs`)
  optimized for network transmission

### Proof Properties

- Proof of inclusion: key exists with this value
- Proof of absence: key does not exist in this range
- Multi-key proofs: single proof covers multiple keys across subtrees
- Proof size is O(log n) per proven key

## Query System

### PathQuery Structure

```rust
PathQuery {
    path: Vec<Vec<u8>>,           // Starting subtree path
    query: SizedQuery {
        query: Query {
            items: Vec<QueryItem>,               // What to select (keys, ranges)
            default_subquery_branch: SubqueryBranch,  // Default descent into subtrees
            conditional_subquery_branches: BTreeMap,   // Key-specific descent rules
            left_to_right: bool,                 // Iteration direction
            add_parent_tree_on_subquery: bool,   // v2: include parent in results
        },
        limit: Option<u16>,       // Max results
        offset: Option<u16>,      // Skip first N results
    }
}
```

### Query Features

- **QueryItem types**: `Key(bytes)`, `Range(start..end)`,
  `RangeInclusive(start..=end)`, `RangeFull(..)`, etc.
- **Subquery branches**: when a query hits a `Tree` element, it can
  automatically descend with a subquery. `default_subquery_branch` applies
  to all trees; `conditional_subquery_branches` applies only to specific keys.
- **`add_parent_tree_on_subquery` (v2)**: when true, includes the parent
  tree element (e.g., a `CountTree` or `SumTree`) in results alongside the
  subquery results. Useful when you need both the aggregate value and the
  individual elements.
- **Cross-subtree queries**: a single PathQuery can traverse multiple levels
  of the grove hierarchy, collecting results from different subtrees
- **Limit/offset**: pagination support applied across the entire result set
- **Direction**: `left_to_right` controls iteration order (ascending vs
  descending keys)

## Batch Operations

Batch processing is the preferred way to perform multiple mutations
atomically (`grovedb/src/batch/mod.rs` in older versions, now integrated
with MerkCache):

### Two-Phase Processing

1. **Validation phase**: all operations are validated against current state
   — checks for conflicts, validates references, ensures tree existence
2. **Application phase**: validated operations are applied atomically

### MerkCache and TreeCache

- `MerkCache` (`grovedb/src/merk_cache.rs`) keeps opened subtrees in memory
  during batch processing to avoid redundant opens
- Deferred root hash propagation: root hashes aren't recomputed after each
  individual operation — instead, dirty subtrees are marked and hashes
  propagate upward only once at commit time
- **Atomic cross-subtree operations**: either all operations across all
  affected subtrees succeed, or none do. The underlying RocksDB transaction
  ensures atomicity.

### Batch Best Practices

- Always prefer batch operations over individual insert/delete calls for
  multiple mutations — significantly reduces disk I/O and hash recomputation
- Batch operations support transient operations (intermediate states that
  don't need to be persisted individually)
- Estimated cost mode allows pre-computing operation costs without execution

## Merk Internals

### TreeNodeInner Structure

```rust
pub struct TreeNodeInner {
    left: Option<Link>,    // Left child link
    right: Option<Link>,   // Right child link
    kv: KV,                // Key-value pair with hashes and feature type
}
```

Every node stores a key-value pair, left/right child links, and a
`TreeFeatureType` that carries aggregate data for specialized trees.

### Link System — Four States

Links represent references to child nodes with different memory/state
characteristics:

1. **`Reference`** — child is NOT in memory; only stores `hash`,
   `child_heights`, `key`, and `aggregate_data`. The child can be fetched
   from RocksDB by key when needed. This is the pruned/lazy state.

2. **`Modified`** — child IS in memory and has been changed since the last
   hash computation. Stores `pending_writes` count and `child_heights` but
   NOT the hash (it's outdated). The hash will be recomputed during commit.

3. **`Uncommitted`** — child IS in memory, has been modified since last
   commit, but its hash IS up-to-date (recently recomputed). Stores `hash`,
   `child_heights`, `tree`, and `aggregate_data`. Ready to be written to
   storage.

4. **`Loaded`** — child IS in memory, has NOT been modified, and has a
   current hash. This is the clean, in-memory cached state. Stores all
   fields.

**Lifecycle**: `Reference` → (fetch from storage) → `Loaded` → (modify) →
`Modified` → (recompute hash) → `Uncommitted` → (commit to storage) →
`Reference`

### AVL Balancing

- Balance factor = `right_height - left_height`
- Must maintain factor ∈ `{-1, 0, 1}` at all times
- Single rotation when imbalance is same direction as heavy child
- Double rotation when imbalance is opposite direction
- Rebalancing propagates up from the modification point
- O(log n) operations guaranteed

### Walker Pattern

The `Walker` (`merk/src/tree/walk/`) provides lazy traversal:
- Only loads child nodes when the traversal actually visits them
- Uses the `Fetch` trait to abstract over storage backends
- `RefWalker` for read-only traversal, `Walker` for mutable access

## Key Files

### Core GroveDB
- `grovedb/src/lib.rs` — public API, doc examples, module structure
- `grovedb/src/operations/insert/mod.rs` — insert logic with element validation
- `grovedb/src/operations/delete/mod.rs` — delete operations including `delete_up_tree` for cascading
- `grovedb/src/operations/get/` — get operations, `MAX_REFERENCE_HOPS` constant
- `grovedb/src/operations/proof/generate.rs` — multi-tree proof generation
- `grovedb/src/operations/proof/` — proof verification (with `verify` feature)
- `grovedb/src/reference_path.rs` — reference resolution with cycle detection
- `grovedb/src/merk_cache.rs` — MerkCache for keeping subtrees open during batch ops
- `grovedb/src/error.rs` — comprehensive error enum (30+ variants)
- `grovedb/src/operations/commitment_tree.rs` — commitment tree operations
- `grovedb/src/operations/mmr_tree.rs` — Merkle mountain range tree operations
- `grovedb/src/operations/bulk_append_tree.rs` — bulk append optimizations
- `grovedb/src/operations/dense_tree.rs` — dense fixed-size tree operations

### Merk Implementation
- `merk/src/tree/mod.rs` — `TreeNodeInner` struct, AVL core
- `merk/src/tree/link.rs` — `Link` enum with 4 states (Reference/Modified/Uncommitted/Loaded)
- `merk/src/tree/walk/mod.rs` — Walker pattern for lazy loading
- `merk/src/tree/ops.rs` — tree operations (put, delete, batch apply)
- `merk/src/tree/commit.rs` — commit logic, hash recomputation
- `merk/src/tree/hash.rs` — Blake3 hashing, node_hash, kv_hash, value_hash
- `merk/src/tree/kv.rs` — KV struct with value_defined_cost types
- `merk/src/tree/tree_feature_type.rs` — `TreeFeatureType`, `AggregateData`
- `merk/src/proofs/query/mod.rs` — query execution and proof generation
- `merk/src/proofs/encoding.rs` — proof serialization
- `merk/src/owner.rs` — reference counting wrapper

### Storage Abstraction
- `storage/src/lib.rs` — `Storage`, `StorageContext`, `StorageBatch` traits
- `storage/src/rocksdb_storage/storage.rs` — RocksDB impl, column families, prefix generation
- `storage/src/rocksdb_storage/storage_context.rs` — prefixed contexts for subtree isolation
- `storage/src/worst_case_costs.rs` — worst-case cost estimation

### Supporting Crates
- `costs/` — `OperationCost`, `StorageCost`, cost macros
- `grovedb-element/` — `Element` enum, serialization, reference path types
- `grovedb-query/` — `Query`, `QueryItem`, `PathQuery` types
- `grovedb-version/` — version management, feature gating macros
- `path/` — `SubtreePath`, `SubtreePathBuilder` utilities

## Error Handling

### Error Types (`grovedb/src/error.rs`)

Major error categories:

- **Reference errors**: `CyclicReference`, `ReferenceLimit`, `MissingReference`,
  `CorruptedReferencePathKeyNotFound`, `CorruptedReferencePathNotFound`
- **Path errors**: `PathKeyNotFound`, `PathNotFound`,
  `PathParentLayerNotFound` — may represent valid queries where data doesn't
  exist
- **Corruption errors**: `CorruptedData`, `CorruptedStorage`,
  `CorruptedCodeExecution` — indicate internal consistency failures
- **Input validation**: `InvalidInput`, `InvalidQuery`, `InvalidPath`,
  `InvalidParameter`, `MissingParameter`
- **Operation errors**: `InvalidBatchOperation`, `DeletingNonEmptyTree`,
  `ClearingTreeWithSubtreesNotAllowed`, `OverrideNotAllowed`
- **Client callback errors**: `JustInTimeElementFlagsClientError`,
  `SplitRemovalBytesClientError` — errors from client-provided callbacks
- **Upstream errors**: `StorageError`, `MerkError`, `VersionError`,
  `ElementError`, `QueryError`

### Error Handling Patterns

```rust
// Accumulate costs through early returns
cost_return_on_error!(&mut cost, result);

// Wrap errors with context
.map_err(|e| Error::CorruptedData(format!("context: {}", e)))?;

// Version-gated functions
check_grovedb_v0_with_cost!("function_name", version_path);
```

## Testing Philosophy

1. **Proof verification**: every state-modifying operation must be testable
   with cryptographic proofs — insert data, generate proof, verify
   independently
2. **Cost accuracy**: tests verify that computed costs match actual storage
   operations. Cost divergence = consensus failure in production.
3. **Reference integrity**: tests ensure references resolve correctly, cycles
   are detected, hop limits are enforced, and dangling references are caught
4. **Version compatibility**: tests run against multiple `GroveVersion`
   variants to ensure backward compatibility across protocol upgrades
5. **Batch atomicity**: tests verify all-or-nothing semantics — partial
   batch failures must roll back completely
6. **Edge cases**: empty trees, maximum depth paths, boundary key values,
   concurrent transaction conflicts

## Development Patterns

### Adding New Features

1. Check `GroveVersion` compatibility — new features need version gating via
   `check_grovedb_v0_with_cost!` or similar macros
2. Implement cost calculation alongside functionality — costs are not
   optional, they're consensus-critical
3. Ensure proof generation works correctly for the new feature
4. Add batch operation support if applicable
5. Write comprehensive tests including edge cases and cost verification

### Performance Considerations

1. Use batch operations for multiple changes — avoids per-operation hash
   propagation overhead
2. Leverage `MerkCache` for repeated access to the same subtrees within a
   transaction
3. Minimize tree opens by using persistent storage contexts
4. Consider cost limits for expensive operations to prevent resource
   exhaustion
5. Use lazy loading — don't force-load entire subtrees when only a few keys
   are needed
6. Estimated costs (feature-gated) allow pre-flight cost checks without
   executing

### Debugging Tips

1. **Visualizer**: `db.start_visualizer(port)` launches a web-based tree
   inspector for examining grove structure
2. **Cost analysis**: log `OperationCost` values to identify expensive
   operations and optimization targets
3. **Proof verification**: test proofs independently to isolate whether the
   issue is in generation or verification
4. **Reference tracing**: manually follow reference chains to debug
   resolution failures; check for cycles
5. **Version checks**: ensure the correct `GroveVersion` is threaded through
   all call sites

## Workspace Structure

```
grovedb/
├── grovedb/                          # Core database — grove management, operations, proofs
├── merk/                             # Merkle AVL tree engine
├── storage/                          # RocksDB storage abstraction
├── costs/                            # OperationCost types and cost macros
├── path/                             # SubtreePath and SubtreePathBuilder
├── grovedb-element/                  # Element enum, reference path types
├── grovedb-query/                    # Query, QueryItem, PathQuery types
├── grovedb-version/                  # Version management and feature gating
├── grovedb-epoch-based-storage-flags/# Epoch-aware storage flag utilities
├── grovedb-bulk-append-tree/         # Bulk append tree operations
├── grovedb-commitment-tree/          # Commitment tree operations
├── grovedb-dense-fixed-sized-merkle-tree/  # Dense fixed-size Merkle tree
├── grovedb-merkle-mountain-range/    # MMR implementation
├── grovedbg-types/                   # Types shared with the grovedbg debugger
├── visualize/                        # Debug visualization tools
├── tutorials/                        # Usage examples
└── docs/                             # Detailed per-crate documentation
    └── crates/
```

## Build & Test

```sh
cargo build
cargo test
cargo test -p grovedb          # Core crate only
cargo test -p merk             # Merk crate only
cargo test -p storage          # Storage crate only
cargo test test_name           # Specific test
cargo test --features full,estimated_costs  # With all features
cargo clippy -- -D warnings
cargo bench                    # Benchmarks
```
