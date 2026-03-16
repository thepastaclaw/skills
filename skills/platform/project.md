# Dash Platform — Project Understanding

## Overview

Dash Platform (`dashpay/platform`) is a monorepo containing ~44 Rust
crates and several JS/TS packages. It implements the Layer 2
decentralized application platform for the Dash network, providing
identity management, data contracts, document storage, and a gRPC
API for clients.

## Architecture

Two architectural pillars:

- **Drive** — state machine built on GroveDB. Stores and manages all
  Platform state (identities, data contracts, documents).
- **DAPI** — decentralized gRPC API layer. Clients connect to any
  masternode's DAPI endpoint to interact with Platform.

## Core Dependency Chain

```
dpp → drive → drive-abci → dash-sdk
```

**Dependency rules (violations are blocking):**
- DPP never imports Drive
- Drive never imports Drive-ABCI
- Never enable the `server` feature of Drive in client crates

## Crate Breakdown

### DPP (Dash Platform Protocol)
- Data model: Identity, DataContract, Document, StateTransition
- Storage-agnostic — defines types without knowing how they're stored
- 80+ feature flags for conditional compilation

### Drive
- Storage engine wrapping GroveDB
- Feature split: `server` (full node operations) vs `verify`
  (proof verification only)

### Drive-ABCI
- ABCI application server for Tenderdash (CometBFT fork)
- Block processing, state transition validation, gRPC queries
- Organized into `abci/`, `execution/`, `query/`

### dash-sdk
- Client SDK for interacting with Platform
- Uses Drive with `verify` feature only — never `server`

### platform-version
- Versioning backbone for consensus-critical methods
- Every consensus-critical method has a version number dispatched via
  `PlatformVersion` struct
- Currently 12 protocol versions

## External Dependencies

- **GroveDB** (`dashpay/grovedb`) — Merkle-tree authenticated data
  structure on RocksDB. Hierarchical, authenticated, transactional,
  cost-tracking. See grovedb skill for details.
- **Tenderdash** — CometBFT fork used for consensus

## System Contracts

- DPNS (Dash Platform Name Service)
- DashPay (social/contact features)
- Withdrawals
- Masternode Reward Shares
- Feature Flags
- Wallet Utils
- Token History
- Keyword Search

## Build & Test

```sh
yarn setup               # Initial setup
cargo test --workspace   # Run all Rust tests
cargo clippy --workspace # Lint
```

## Conventions

- **PR format:** Conventional Commits, template in
  `.github/PULL_REQUEST_TEMPLATE.md`
- **Rust edition:** 2021
- **MSRV:** 1.92
- **Current version:** 3.0.1

## Key Maintainers

- **QuantumExplorer** — Platform lead
- **shumkov** — Core developer
- **lklimek** — Core developer
- **okuznetsov** — Core developer
- **HashEngineering** — Core developer
