# GroveDB — Project Understanding

## Overview

GroveDB (`dashpay/grovedb`) is a hierarchical authenticated data
structure database — a "grove" (tree of trees) built on Merkle AVL
trees backed by RocksDB. It provides cryptographic proofs for all
queries, enabling trustless verification of state.

## Architecture — Three Layers

```
GroveDB core → Merk → Storage
```

1. **GroveDB core** — manages the grove (tree of subtrees), handles
   path-based operations, batch processing, proofs, and queries
2. **Merk** — Merkle AVL tree implementation with lazy loading via
   a Link system
3. **Storage** — RocksDB abstraction with Blake3-hashed prefixes
   for subtree isolation

## Crate Structure

- `grovedb` — core library
- `merk` — Merkle AVL tree
- `storage` — RocksDB abstraction
- `costs` — cost tracking types
- `path` — path utilities
- `grovedb-version` — version management
- `visualize` — tree visualization tools

## Element System

8 element types:
- `Item` — raw key-value
- `Reference` — pointer to another element
- `Tree` — subtree root (creates hierarchy)
- `SumItem` — item contributing to a sum
- `SumTree` — subtree that tracks sum of SumItems
- `BigSumTree` — large-value sum tree
- `CountTree` — subtree that counts elements
- `CountSumTree` — subtree that counts and sums

## Reference System

7 reference types for flexible cross-tree pointers:
- AbsolutePath, UpstreamRootHeight, UpstreamFromElementHeight,
  Cousin, Sibling, and others
- Hop limits prevent infinite resolution chains
- Cycle detection required

## Cost Tracking

Every operation tracks costs:
- `seek_count` — number of disk seeks
- `storage_loaded_bytes` — bytes read from storage
- `storage_cost` — storage write costs
- `hash_node_calls` — hashing operations

Cost accuracy is critical — Platform consensus depends on all nodes
computing identical costs.

## Proof System

- Layer-by-layer proof generation
- Stack-based verification
- Proofs cover entire query results with cryptographic guarantees

## Query System

- `PathQuery` with `SizedQuery` for bounded queries
- Conditional subquery branches for complex traversals
- Queries can span multiple subtrees

## Batch Operations

- Two-phase processing: validation then application
- Atomic across subtrees — all-or-nothing
- Preferred over individual operations for performance

## Storage Layer

- RocksDB backend
- Blake3-hashed prefixes isolate subtrees
- 4 storage types for different data categories
- Prefix isolation prevents cross-subtree data leakage

## Merk Internals

- AVL tree with balance factor always in {-1, 0, 1}
- Link system with 4 states: Reference, Modified, Uncommitted, Loaded
- Lazy loading — only loads nodes when accessed
- MerkCache for frequently accessed subtrees

## Build & Test

```sh
cargo build
cargo test
cargo clippy -- -D warnings
```
