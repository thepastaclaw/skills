# Dash Core — Project Understanding

## Overview

Dash Core (`dashpay/dash`) is a fork of Bitcoin Core implementing
the Dash cryptocurrency. It extends Bitcoin with masternodes,
InstantSend, ChainLocks, CoinJoin, and a decentralized governance
system. It also serves as the L1 layer for Dash Platform (L2).

## Source Directory Structure

```
src/
├── consensus/          # Consensus parameters, validation rules
├── coinjoin/           # CoinJoin mixing (client + server)
├── evo/                # Evolution layer — special TXs, deterministic MN list
├── governance/         # Governance proposals, voting, superblocks
├── llmq/               # Long-Living Masternode Quorums
├── masternode/         # Masternode management, PoSe, payments
├── index/              # Database indexes (BaseIndex framework)
├── net_processing.cpp  # P2P message handling
├── validation.cpp      # Block/transaction validation (consensus-critical)
├── init.cpp            # Node initialization and shutdown
├── rpc/                # RPC interface
├── wallet/             # Wallet functionality
├── qt/                 # Qt GUI
├── test/               # Unit tests
├── interfaces/         # Abstract interfaces (node, wallet, chain)
├── crypto/             # Cryptographic primitives
├── bls/                # BLS signature scheme (legacy + basic)
├── script/             # Script evaluation
├── primitives/         # Core types (CTransaction, CBlock)
└── Makefile.am         # Build configuration (add new files here!)

test/
├── functional/         # Python functional tests
├── util/               # Test utilities
└── lint/               # Linting scripts
```

## Dash-Specific Subsystems

### Evolution Layer (`src/evo/`)
- **Special Transactions:** Dash extends Bitcoin's transaction
  format with typed transactions (ProRegTx, ProUpServTx,
  ProUpRegTx, ProUpRevTx, CoinbaseTx, QuorumCommitmentTx)
- **Deterministic Masternode List:** `CDeterministicMNManager`
  manages the authoritative list of masternodes derived from
  on-chain ProTx transactions
- **Credit Pool:** Bridge between L1 (Dash Core) and L2 (Platform)
  via pool-based accounting for asset lock/unlock
- **MN Auth:** Masternode authentication for P2P connections

### LLMQ Subsystem (`src/llmq/`)
- **Quorum Types:**
  - `LLMQ_60_75` (60 members, 75% threshold) — InstantSend
  - `LLMQ_400_60` (400 members, 60%) — ChainLocks
  - `LLMQ_400_85` (400 members, 85%) — hard fork signaling
  - `LLMQ_100_67` (100 members, 67%) — Platform consensus (evonodes)
- **DKG:** Distributed Key Generation for quorum formation
- **Signing:** Threshold BLS signing for InstantSend locks and
  ChainLocks
- **ChainLocks:** Lock blocks to prevent 51% attacks
- **InstantSend:** Instant transaction confirmation via quorum
  signing
- **Quorum Rotation:** Quarter-based rotation (DIP-24) prevents
  double-signing during transitions

### CoinJoin (`src/coinjoin/`)
- Mixing protocol for transaction privacy
- Client (mixing participant) and server (masternode) components
- Denomination-based mixing rounds

### Governance (`src/governance/`)
- Decentralized governance with proposals and voting
- Superblocks for budget payments
- Masternode voting (one vote per masternode)

### Masternode Management (`src/masternode/`)
- Masternode payments and payment ordering
- Proof of Service (PoSe) — ban masternodes for misbehavior
- Masternode sync protocol
- Two types: Regular (1000 DASH collateral) and Evonode (4000
  DASH, runs Platform)

## Key Types and Classes

- `CDeterministicMN` / `CDeterministicMNList` — masternode state
- `CQuorum` — LLMQ quorum instance
- `CTransaction` / `CMutableTransaction` — transactions
- `CBlock` / `CBlockIndex` — blocks and chain index
- `CConnman` / `CNode` — P2P networking
- `CTxMemPool` — transaction mempool
- `CChainState` — UTXO set and chain state
- `CGovernanceManager` — governance object management
- `CCoinJoinClientManager` — CoinJoin client logic

## BLS Scheme

Dash uses two BLS signature schemes:
- **Legacy** — original scheme (pre-V19)
- **Basic** — introduced in V19 deployment
- The global flag `bls::bls_legacy_scheme` controls which is active
- Source of subtle bugs when schemes are mixed

## Build System

- **Autotools** (autoconf/automake) — `./autogen.sh && ./configure && make`
- **Depends system** (`depends/`) — Makefile-based cross-compilation
  for all target platforms
- **Guix** (`contrib/guix/`) — reproducible deterministic builds
- New source files MUST be added to `src/Makefile.am`

## Testing

- **Unit tests:** `src/test/` — C++ Boost.Test framework.
  Run with `make check`
- **Functional tests:** `test/functional/` — Python framework.
  `DashTestFramework` extends Bitcoin Core's `BitcoinTestFramework`.
  Run with `test/functional/test_runner.py`
- **Test activation:** `-testactivationheight=v19@200` for
  deployment testing

## Deployment / Feature Activation

- Named deployments: `DEPLOYMENT_V19`, `DEPLOYMENT_V20`, etc.
- `DeploymentActiveAfter()` / `DeploymentActiveAt()` check
  activation
- BIP9-style signaling via version bits

## Bitcoin Core Upstream Relationship

- Dash Core is based on Bitcoin Core with periodic backports
- Backported changes preserve original authorship
- `test/util/data/non-backported.txt` tracks intentionally skipped
  upstream changes
- Dash-specific code is interleaved with Bitcoin Core code — not
  cleanly separated

## Key Maintainers

- **PastaPastaPasta** — Core maintainer, broad ownership
- **UdjinM6** — Core developer, masternode/LLMQ expertise
- **QuantumExplorer** — Platform lead (GroveDB, Drive)
- **knst** — Core developer, backports, LLMQ

## Project Direction

- Ongoing Bitcoin Core backports for security and features
- Platform integration (credit pool, asset locks, Rust bindings)
- Evonode functionality expansion
- Performance and scalability improvements
