# GroveDB — Core Review Context

Always-loaded architectural context for reviewing `dashpay/grovedb`: the
invariants a reviewer needs on *every* diff — the three-layer architecture,
elements and references, cost tracking, proofs, queries, batching, Merk
internals, version gating. Crate file maps, testing philosophy,
performance/debugging notes, and build commands live in `reference.md` and are
loaded on demand.

Facts checked against the `develop` checkout at commit `32ae393b` (2026-03-06).
Counts carry an as-of marker; when a count matters to a finding, re-read the
cited source rather than trusting this document.

## Overview

GroveDB is a hierarchical authenticated data structure — a "grove" (tree of
trees) built on Merkle AVL trees backed by RocksDB. Its distinguishing property
is that it manages **multiple Merk trees arranged hierarchically**: any element
can itself be a subtree root, creating an arbitrarily deep grove. That enables
path-based addressing (e.g. `["identities", id, "keys"]`) and composite proofs
spanning multiple subtrees in one response. Every query result can be verified
independently against a root hash, without trusting the data source.

GroveDB runs on every Dash Platform masternode. Consensus requires all nodes to
produce **identical root hashes and identical costs** for the same operations,
so hashing, proof encoding, and cost accounting are consensus-critical surfaces,
not implementation details.

## Three-Layer Architecture

```
GroveDB Core (grovedb/)  →  Merk (merk/)  →  Storage (storage/)
```

**Layer 1 — GroveDB Core (`grovedb/src/`)** orchestrates multiple Merk trees
into the grove: path-based insert/get/delete addressed by `(path, key)` where
`path` is a vector of byte-string segments; element management; reference
resolution with hop limits and cycle detection; batch processing via two-phase
validation-then-application; composite proof generation; `PathQuery` execution
with conditional subquery branches. `MerkCache` (`grovedb/src/merk_cache.rs`)
keeps frequently accessed subtrees open in memory — it uses `UnsafeCell`
internally for interior mutability with a per-Merk borrow-flag guard, so changes
there need scrutiny for aliasing.

**Layer 2 — Merk (`merk/src/`)** is a Merkle AVL tree; each subtree in the grove
is a Merk instance.
- AVL self-balancing: balance factor always in `{-1, 0, 1}`, enforced via
  single/double rotations → O(log n) guaranteed
- Intermediary nodes store key-value pairs — unlike typical Merkle trees, every
  node holds data, not just leaves
- Lazy loading via the Link system (below): nodes load from storage on access
- Every operation accumulates an `OperationCost`
- Chunk-based restoration lets large trees restore incrementally for
  replication/sync
- Aggregate tree types have predefined value sizes, enabling cost prediction

**Layer 3 — Storage (`storage/src/`)** abstracts RocksDB with prefixed storage
for subtree isolation.
- `OptimisticTransactionDB` backend for concurrent transactions
- **Blake3 prefix isolation**: each subtree's path segments hash to a 32-byte
  prefix; all its keys live under that prefix, preventing cross-subtree leakage.
  Prefix generation must stay deterministic.
- **Four column families**: `default` (tree node data), `aux` (auxiliary data
  per subtree), `roots` (subtree root node metadata), `meta` (database-level
  metadata) — `AUX_CF_NAME` / `ROOTS_CF_NAME` / `META_CF_NAME` in
  `storage/src/rocksdb_storage/storage.rs`
- `StorageBatch` accumulates writes and flushes them in one RocksDB write batch
- `PrefixedRocksDbTransactionContext` and
  `PrefixedRocksDbImmediateStorageContext` scope reads/writes to one prefix

## Element System

`Element` (`grovedb-element/src/element/mod.rs`) — 15 variants as of the
reference commit. The set has grown well past the original eight; check the enum
before asserting a variant list is complete.

| Group | Variants |
|-------|----------|
| Plain data | `Item`, `ItemWithSumItem` |
| Pointer | `Reference` |
| Subtree | `Tree` |
| Sum | `SumItem`, `SumTree`, `BigSumTree` |
| Count | `CountTree`, `CountSumTree`, `ProvableCountTree`, `ProvableCountSumTree` |
| Specialized trees | `CommitmentTree`, `MmrTree`, `BulkAppendTree`, `DenseAppendOnlyFixedSizeTree` |

Aggregate tree types store their aggregate in the Merk node's feature type field
(`TreeFeatureType` / `AggregateData`), which propagates upward during
rebalancing — so aggregates are available at the root without scanning
descendants. That propagation is the thing to check on any rotation, insert, or
delete change: an aggregate that fails to propagate produces a wrong root hash on
some nodes and not others.

Adding a variant touches serialization, cost prediction, proof encoding, and
version gating together. A change that adds one without all four is incomplete.

## Reference System

`ReferencePathType` (`grovedb-element/src/reference_path/mod.rs`) — 7 variants:

1. **AbsolutePathReference** — full path from root; simple, brittle if the layout
   changes
2. **UpstreamRootHeightReference** — up to height N from root, then a relative path
3. **UpstreamRootHeightWithParentPathAdditionReference** — up to a root height,
   append the parent's path, then follow additional segments
4. **UpstreamFromElementHeightReference** — up N levels from the current
   element's position, then a path
5. **CousinReference** — same depth, different branch (sibling of the parent)
6. **RemovedCousinReference** — cousin form where the cousin key is
   removed/replaced rather than appended
7. **SiblingReference** — same parent tree, different key; the simplest relative form

Resolution runs through `follow_reference` in `grovedb/src/reference_path.rs`
(public entry point `GroveDb::follow_reference` in
`grovedb/src/operations/get/mod.rs`). `MAX_REFERENCE_HOPS` (defined in
`operations/get`) bounds chains, yielding `Error::ReferenceLimit`; a `HashSet` of
visited qualified paths returns `Error::CyclicReference` on repeats. A reference
pointing at another reference keeps resolving until it reaches a non-reference
element or hits the limit; `follow_reference_once` stops at the first resolved
element.

Any new resolution path must enforce both the hop limit and cycle detection, and
must accumulate cost for every hop — a hop that resolves for free is a free
denial-of-service lever.

## Cost Tracking

```rust
pub struct OperationCost {
    pub seek_count: u32,             // RocksDB seeks (disk I/O)
    pub storage_cost: StorageCost,   // added_bytes / replaced_bytes / removed_bytes
    pub storage_loaded_bytes: u64,   // bytes read from storage
    pub hash_node_calls: u32,        // Blake3 node hashes
    pub sinsemilla_hash_calls: u32,  // Sinsemilla (elliptic-curve) hashes for
                                     // commitment tree anchors — far more
                                     // expensive than Blake3
}
```
`StorageCost { added_bytes: u32, replaced_bytes: u32, removed_bytes: StorageRemovedBytes }`.

Any divergence in computed cost between nodes is a consensus failure and a chain
halt. So: cost calculation is on the consensus-critical path, not an
observability feature; tests must verify cost accuracy against actual operations;
estimated costs (`estimated_costs` feature) must match what execution charges;
and Sinsemilla calls must be counted separately from Blake3 — folding them into
`hash_node_calls` undercharges commitment-tree work.

```rust
cost_return_on_error!(&mut cost, some_operation());          // accumulate, then return on error
cost_return_on_error_no_add!(&mut cost, some_operation());   // don't add on the error path
cost_return_on_error_into_no_add!(cost, some_operation());   // Into conversion variant
```
A bare `?` on a cost-returning call where one of these belongs silently drops
accumulated cost. Flag it.

## Proof System

**Generation** (`grovedb/src/operations/proof/generate.rs`) works layer by
layer: start at the root tree, descend to target subtrees, generate a Merk proof
at each layer via depth-first traversal collecting nodes along the path. Proofs
are minimal — only nodes on the path to queried keys plus what is needed to
recompute the root. Absence proofs prove a key does NOT exist by including the
neighbouring nodes that bracket it.

**Verification** (`grovedb/src/operations/proof/`, `verify` feature) is
stack-based: operations (push node, hash combine, …) execute sequentially against
a stack. A verifier needs only the proof bytes and the expected root hash, never
the database. Proof encoding lives under `merk/src/proofs/` (`mod.rs`,
`tree.rs`, `query/`) and is optimized for network transmission.

Properties: inclusion (key exists with this value), absence (key not in range),
multi-key (one proof covers several keys across subtrees), size O(log n) per
proven key.

Generation and verification must move together. A change to one side without the
matching change to the other produces proofs that verify on the author's branch
and fail everywhere else — always blocking.

## Query System

```rust
PathQuery {
    path: Vec<Vec<u8>>,           // Starting subtree path
    query: SizedQuery {
        query: Query {
            items: Vec<QueryItem>,                     // keys, ranges
            default_subquery_branch: SubqueryBranch,   // default descent into subtrees
            conditional_subquery_branches: BTreeMap,   // key-specific descent rules
            left_to_right: bool,                       // iteration direction
            add_parent_tree_on_subquery: bool,         // v2: include parent in results
        },
        limit: Option<u16>,
        offset: Option<u16>,
    }
}
```

- **QueryItem** forms: `Key(bytes)`, `Range(start..end)`,
  `RangeInclusive(start..=end)`, `RangeFull(..)`, and relatives
- **Subquery branches**: when a query hits a `Tree` element it can descend
  automatically. `default_subquery_branch` applies to all trees;
  `conditional_subquery_branches` only to specific keys.
- **`add_parent_tree_on_subquery` (v2)**: includes the parent tree element (e.g.
  a `CountTree` or `SumTree`) alongside subquery results, when both the aggregate
  and the individual elements are needed
- One PathQuery can traverse multiple grove levels; limit/offset paginate across
  the entire result set; `left_to_right` controls iteration order

Query semantics feed proof generation. A query change that alters which elements
are returned also alters what the proof covers, so the two must be reviewed
together — and limit/offset interactions with subqueries are a recurring source
of proof/result mismatches.

## Batch Operations

Batching is the preferred way to perform multiple mutations atomically, in two
phases: **validation** (all operations checked against current state — conflicts,
reference validity, tree existence) then **application** (validated operations
applied atomically).

- `MerkCache` keeps opened subtrees in memory during batch processing, avoiding
  redundant opens
- **Deferred root hash propagation**: hashes are not recomputed after each
  operation; dirty subtrees are marked and hashes propagate upward once at commit
  time. A path that mutates a subtree without marking it dirty produces a stale
  root hash.
- **Atomic cross-subtree operations**: all operations across all affected
  subtrees succeed or none do; the RocksDB transaction provides atomicity
- Prefer batches over individual insert/delete for multiple mutations —
  materially less disk I/O and hash recomputation
- Batches support transient operations (intermediate states not persisted
  individually), and estimated-cost mode allows pre-computing costs without
  executing
- Partial failure must roll back completely; a batch that leaves some subtrees
  mutated is state corruption

## Merk Internals

```rust
pub struct TreeNodeInner {
    left: Option<Link>,    // Left child link
    right: Option<Link>,   // Right child link
    kv: KV,                // Key-value pair with hashes and feature type
}
```

### Link System — Four States
`Link` (`merk/src/tree/link.rs`):

1. **`Reference`** — child NOT in memory; stores `hash`, `child_heights`, `key`,
   `aggregate_data`; fetched from RocksDB on demand. The pruned/lazy state.
2. **`Modified`** — child IS in memory and changed since the last hash
   computation. Stores `pending_writes` and `child_heights` but NOT a hash (it
   would be stale). Hash is recomputed at commit.
3. **`Uncommitted`** — child IS in memory, modified since last commit, hash IS up
   to date. Ready to write.
4. **`Loaded`** — child IS in memory, unmodified, hash current. The clean cached
   state.

**Lifecycle:** `Reference` → (fetch) → `Loaded` → (modify) → `Modified` →
(recompute hash) → `Uncommitted` → (commit) → `Reference`. Transitions that skip
a state — particularly reading a hash out of a `Modified` link, or committing
without passing through `Uncommitted` — write a wrong hash to storage.

### AVL Balancing
Balance factor = `right_height - left_height`, always in `{-1, 0, 1}`. Single
rotation when the imbalance matches the heavy child's direction, double rotation
when opposite. Rebalancing propagates up from the modification point. Rotations
must preserve sorted order, update hashes, and update aggregate data.

### Walker Pattern
`Walker` (`merk/src/tree/walk/`) provides lazy traversal: children load only when
the traversal visits them, via the `Fetch` trait abstraction over storage
backends. `RefWalker` is read-only; `Walker` allows mutation.

## Error Handling

`Error` (`grovedb/src/error.rs`) — 41 variants as of the reference commit,
grouped by cause:

- **Reference**: `CyclicReference`, `ReferenceLimit`, `MissingReference`,
  `CorruptedReferencePathKeyNotFound`, `CorruptedReferencePathNotFound`
- **Path**: `PathKeyNotFound`, `PathNotFound`, `PathParentLayerNotFound` — these
  can represent *valid* queries for data that does not exist, so treat them as
  expected outcomes rather than failures unless the caller says otherwise
- **Corruption**: `CorruptedData`, `CorruptedStorage`, `CorruptedCodeExecution` —
  internal consistency failures, i.e. bugs
- **Input validation**: `InvalidInput`, `InvalidQuery`, `InvalidPath`,
  `InvalidParameter`, `MissingParameter`
- **Operation**: `InvalidBatchOperation`, `DeletingNonEmptyTree`,
  `ClearingTreeWithSubtreesNotAllowed`, `OverrideNotAllowed`
- **Client callback**: `JustInTimeElementFlagsClientError`,
  `SplitRemovalBytesClientError`
- **Upstream**: `StorageError`, `MerkError`, `VersionError`, `ElementError`,
  `QueryError`

Collapsing a specific variant into a generic one (returning `CorruptedData`
where `PathKeyNotFound` is correct) changes caller behavior in Platform, which
branches on these.

```rust
cost_return_on_error!(&mut cost, result);                        // cost-preserving propagation
.map_err(|e| Error::CorruptedData(format!("context: {}", e)))?;  // wrap with context
check_grovedb_v0_with_cost!("function_name", version_path);      // version gating
```

## Version Gating

New behavior must be gated on `GroveVersion` so nodes at an older version keep
producing the old bytes. Macros in `grovedb-version/src/lib.rs`:
`check_grovedb_v0!`, `check_grovedb_v0_with_cost!`, `check_grovedb_v0_or_v1!`,
`check_grovedb_v0_or_v1_with_cost!`.

A behavioral change landed without version gating is a hard fork, regardless of
diff size. `GroveVersion` must be threaded through every call site on the changed
path — a function reaching for a default version instead of the caller's is the
usual way this bug hides.

## Key Conventions for Contributors

- Proof generation and verification change together, or not at all
- Cost accounting is consensus-critical: every new operation, hop, and hash must
  be charged, and estimated cost must match executed cost
- Use the `cost_return_on_error!` family, not bare `?`, on cost-returning calls
- References must always enforce hop limits and cycle detection
- Batches are all-or-nothing; partial application is corruption
- Aggregate data must propagate on every rebalance
- Behavior changes require `GroveVersion` gating and a threaded version argument
- Storage prefix generation must remain deterministic and isolating
