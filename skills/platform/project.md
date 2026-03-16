# Dash Platform — Project Understanding

## Overview

Dash Platform (`dashpay/platform`) is a monorepo of ~44 Rust crates and
several JS/TS packages implementing a Layer 2 decentralized application
platform for Dash. It provides identity management, data contracts,
document storage, token operations, and a gRPC API. The system runs on
masternodes via Tenderdash (a CometBFT fork) and persists state in
GroveDB, a Merkle-tree authenticated database on RocksDB.

## Architecture & Core Dependency Chain

```
dpp → drive → drive-abci → dash-sdk
```

**Dependency rules (violations are blocking):**
- DPP never imports Drive
- Drive never imports Drive-ABCI
- Never enable the `server` feature of Drive in client crates

### Crate Roles

| Role | Crates |
|------|--------|
| **Protocol** | `dpp` (data model), `platform-value` (dynamic values), `platform-serialization` (versioned encoding) |
| **Storage** | `drive` (state machine on GroveDB), `drive-abci` (ABCI app server) |
| **Versioning** | `platform-version`, `platform-versioning` (macros + types), `versioned-feature-core` |
| **Client** | `dash-sdk` (Rust SDK), `wasm-dpp` (WASM bindings), JS SDK packages |
| **Consensus** | `rs-tenderdash-abci` (ABCI interface), signature/key crates |
| **Testing** | `strategy-tests`, `simple-signer`, test helpers |

### DPP (Dash Platform Protocol)
- Defines Identity, DataContract, Document, StateTransition — storage-agnostic
- 80+ feature flags for conditional compilation
- Shared between client and platform (the "protocol library")

### Drive
- Storage engine wrapping GroveDB with domain-specific operations
- Feature split: `server` (full node — reads, writes, proof generation) vs `verify` (proof verification only, used by SDK)
- Contains `state_transition_action` types — the validated, resolved representations that get applied to storage

### Drive-ABCI
- ABCI application server for Tenderdash
- Block processing pipeline, state transition validation, gRPC query handlers
- Organized into `abci/` (Tenderdash interface), `execution/` (block/transition processing), `query/` (gRPC handlers)

### dash-sdk
- Client SDK using Drive with `verify` feature only
- SdkBuilder for configuration; supports normal mode (gRPC to network) and mock mode (local Drive instance for testing)
- Fetch/FetchMany traits for proof-verified queries; PutDocument for submissions

### platform-version
- Versioning backbone — every consensus-critical method has a version number
- Currently 12 protocol versions (v1–v12), stored as static `PlatformVersion` structs in a `PLATFORM_VERSIONS` array
- Looked up by index: `PlatformVersion::get(protocol_version)`

## Versioning System

### PlatformVersion Struct

An immutable snapshot pinning every versioned behavior in the system. Contains nested version sub-structs 5+ levels deep:

```
PlatformVersion
  ├── protocol_version: u32
  ├── drive_abci: DriveAbciVersion
  │     ├── validation_and_processing: DriveAbciValidationVersions
  │     │     └── state_transitions: DriveAbciStateTransitionValidationVersions
  │     │           ├── contract_create_state_transition: DriveAbciStateTransitionValidationVersion
  │     │           │     ├── structure: FeatureVersion
  │     │           │     ├── state: FeatureVersion
  │     │           │     └── transform_into_action: FeatureVersion
  │     │           └── ...
  │     └── ...
  ├── drive: DriveVersion
  │     └── methods: DriveMethodVersions (25+ sub-structs)
  └── ...
```

`FeatureVersion` is a `u16`. `OptionalFeatureVersion` is `Option<u16>` for features introduced in later versions.

### Versioned Dispatch Pattern (Critical)

Every consensus-critical method follows this exact pattern:

```rust
// In mod.rs — the dispatcher
fn update_contract(...) -> Result<(), Error> {
    let version = platform_version.drive.methods.contract.update.update_contract;
    match version {
        0 => self.update_contract_v0(...),
        1 => self.update_contract_v1(...),
        v => Err(Error::UnknownVersionMismatch {
            method: "update_contract".to_string(),
            known_versions: vec![0, 1],
            received: v,
        }),
    }
}
```

**Directory convention:** `mod.rs` (dispatcher) + `v0/mod.rs`, `v1/mod.rs` (implementations).

**Adding a new version:**
1. Create `v{N}/mod.rs` with the new implementation
2. Add the match arm in `mod.rs`
3. Add the version number to the appropriate `PlatformVersion` structs
4. Set the new version number in the latest platform version constant

## State Transitions

### The StateTransition Enum

```rust
pub enum StateTransition {
    DataContractCreate(DataContractCreateTransition),
    DataContractUpdate(DataContractUpdateTransition),
    Batch(BatchTransition),                    // documents + tokens
    IdentityCreate(IdentityCreateTransition),
    IdentityTopUp(IdentityTopUpTransition),
    IdentityCreditWithdrawal(IdentityCreditWithdrawalTransition),
    IdentityUpdate(IdentityUpdateTransition),
    IdentityCreditTransfer(IdentityCreditTransferTransition),
    MasternodeVote(MasternodeVoteTransition),
    // address-based variants (v10+)
    AddressFundsTransfer(...),
    AddressCreditWithdrawal(...),
    AddressDocumentsBatch(...),
    AddressIdentityCreate(...),
    AddressIdentityCreditWithdrawal(...),
}
```

`BatchTransition` aggregates multiple document/token operations into one transition. Address-based variants use UTXO-style input/output instead of identity-based signing.

### Validation Pipeline (10 stages)

Every state transition goes through these stages in order:

1. **is_allowed** — is this transition type enabled in the current protocol version?
2. **signature verification** — verify the transition's cryptographic signature
3. **address witness validation** — for address-based transitions, verify witnesses
4. **address balances/nonces** — for address-based, check inputs have sufficient funds
5. **identity nonces** — for identity-based, verify nonce is correct (prevents replay)
6. **basic structure** — schema-level validation (field types, required fields, sizes)
7. **balance pre-check** — does the identity have enough credits for estimated fees?
8. **advanced structure** — deeper structural validation
9. **transform_into_action** — resolve references against state, produce a `StateTransitionAction`
10. **state validation** — final checks against current platform state

The transition (what the client sends) becomes an action (what the platform executes). Actions live in `rs-drive`, transitions in `rs-dpp`.

### System Actions

When a transition fails validation *after* the nonce check, the platform still needs to bump the nonce and charge fees. System actions handle this:
- `BumpIdentityNonceAction`
- `BumpIdentityDataContractNonceAction`
- `PartiallyUseAssetLockAction`
- `BumpAddressInputNoncesAction`

## Three-Tier Operation Pipeline

```
StateTransitionAction
    │ DriveHighLevelOperationConverter::into_high_level_drive_operations()
    ▼
Vec<DriveOperation>          (domain-level: documents, identities, tokens)
    │ DriveLowLevelOperationConverter::into_low_level_drive_operations()
    ▼
Vec<LowLevelDriveOperation>  (GroveDB-level: insert/delete/update at paths)
    │ apply_batch_low_level_drive_operations()
    ▼
GroveDB (applied atomically)
```

- A single document create can produce 10+ GroveDB operations (one per index plus document plus metadata)
- `into_high_level_drive_operations` is pure — no state reads or writes
- `into_low_level_drive_operations` may read state for cost estimation
- When `apply = false`, the pipeline runs in estimation mode for fee pre-checks

### DriveOperation Enum

```rust
pub enum DriveOperation<'a> {
    DataContractOperation(DataContractOperationType<'a>),
    DocumentOperation(DocumentOperationType<'a>),
    TokenOperation(TokenOperationType),
    IdentityOperation(IdentityOperationType),
    WithdrawalOperation(WithdrawalOperationType),
    SystemOperation(SystemOperationType),
    GroupOperation(GroupOperationType),
    // ... plus raw GroveDB escape hatches
}
```

### LowLevelDriveOperation Enum

```rust
pub enum LowLevelDriveOperation {
    GroveOperation(QualifiedGroveDbOp),     // concrete tree mutation
    FunctionOperation(FunctionOp),           // CPU cost (hashing, sig verify)
    CalculatedCostOperation(OperationCost),  // pre-computed cost
    PreCalculatedFeeResult(FeeResult),       // already-computed fee
}
```

## Data Model

### Data Contracts
- Versioned enum: `DataContract::V0(...)`, `DataContract::V1(...)`
- Accessor traits pattern: `DataContractV0Getters` trait with methods like `id()`, `owner_id()`, `document_types()`
- V1 adds: groups (multi-sig authorization), tokens, timestamps, associated token events, keywords
- Contracts define document types with JSON Schema validation, index rules, and storage configuration
- Contract ID is deterministically computed from owner ID + identity nonce

### Documents
- **Not self-describing** — you need the document type definition from the contract to deserialize
- Deterministic ID: `sha256(contract_id + owner_id + document_type_name + entropy)`
- Fields: `id`, `owner_id`, `revision`, `created_at`, `updated_at`, `transferred_at`, `properties` (BTreeMap)
- Accessor traits: `DocumentV0Getters`, `DocumentV0Setters`

### Identities
- Structure: `id`, `public_keys` (BTreeMap<KeyID, IdentityPublicKey>), `balance` (Credits), `revision`
- Credits: 1 Dash = 100,000,000,000 credits (100 billion)
- Keys have: `purpose` (Authentication/Encryption/Transfer/Voting/Owner), `security_level` (Master/Critical/High/Medium), `key_type` (ECDSA/BLS/EDDSA/BIP13)
- `PartialIdentity` — loads only the fields needed for a specific operation (balance, revision, specific keys)
- Nonce system: 40-bit monotonic counter + 24-bit recent-nonce bitfield for replay protection

## Drive & GroveDB

### GroveDB
- Hierarchical authenticated Merkle tree database on RocksDB
- Subtrees can be nested (paths are vectors of byte-string segments)
- Sum trees track aggregate values (used for balance accounting)
- Fully transactional — all reads/writes in a block happen in one transaction
- Cost-tracking built in — every operation reports its storage and processing cost

### Drive Operations Pattern
- `drive_operations: &mut Vec<LowLevelDriveOperation>` accumulator passed through call chains
- Stateless queries: `DirectQueryType::StatelessDirectQuery` uses estimated layer sizes instead of reading GroveDB
- Stateful queries: `DirectQueryType::StatefulDirectQuery` reads actual GroveDB state
- Finalization tasks: post-commit callbacks (e.g., cache invalidation) via `DriveOperationFinalizeTask`

## Fee System

### Three Fee Eras

| Era | Versions | Authentication | Fee Model |
|-----|----------|---------------|-----------|
| Identity Credits | v1–v9 | Identity keys sign transitions | Deduct credits from identity balance |
| Platform Addresses | v10–v11 | UTXO-style inputs/outputs | Deduct from address inputs |
| Shielded | v12+ | Zero-knowledge proofs | Three-component: proof + per-action processing + per-action storage |

### Fee Components
- **Storage fee:** Proportional to bytes written. Ongoing cost (state occupies tree space indefinitely). Partially refundable when data is deleted.
- **Processing fee:** CPU work — hashing, sig verification, tree traversal. Ephemeral, not refundable.
- `user_fee_increase`: percentage multiplier on processing fee for priority bidding.

### Fee Distribution
- Collected fees go into epoch-based distribution pools
- 50 eras × ~40 epochs each = ~2000 distribution periods
- Proposer gets 100% of processing fees; storage fees distributed over time

## Error Handling

### Consensus Errors (user-facing, cross-node)
- Must be deterministic — all nodes must agree on which error occurred
- Serializable with `PlatformSerialize`/`PlatformDeserialize`
- **DO NOT CHANGE THE ORDER** of enum variants — they are serialized by index
- Four categories with code ranges:
  - `BasicError` (10000–10899): structural/schema failures
  - `SignatureError` (20000–20012): authentication failures
  - `FeeError` (30000+): insufficient balance, fee disputes
  - `StateError` (40000–40899): state conflicts (duplicate keys, missing documents)

### Drive Errors (internal, single-node)
- Not serialized, not consensus-critical
- 13 variants organized by subsystem: `GroveDB`, `Contract`, `Document`, `Identity`, `Fee`, `Proof`, etc.
- Large error types wrapped in `Box<>` to keep the enum small
- Severity spectrum: `CorruptedCodeExecution` (bug), `CorruptedElementType` (data corruption), `NotFound` (missing data)

## Serialization

### Platform Serialization
- Wraps bincode with PlatformVersion-aware encode/decode
- `PlatformVersionEncode` / `PlatformVersionedDecode` traits
- Derive macros: `PlatformSerialize`, `PlatformDeserialize`, `PlatformSignable`
- Key attributes:
  - `#[platform_serialize(limit = 100000)]` — size cap
  - `#[platform_serialize(unversioned)]` — skip version prefix
  - `#[platform_serialize(into = "OtherType")]` — convert before serializing
  - `#[platform_signable(exclude)]` — omit field from signature hash

### Serialization Rules
- All consensus-critical types must use platform serialization (not raw bincode/serde)
- Version prefix enables forward-compatible deserialization
- `PlatformSignable` produces the exact bytes that get signed — `exclude` fields (like the signature itself) are replaced with defaults

## ABCI & Block Processing

### Tenderdash Integration
Four ABCI entry points:
1. **check_tx** — mempool validation (lightweight, no state changes)
2. **prepare_proposal** — proposer builds block content
3. **process_proposal** — validators verify proposed block
4. **finalize_block** — commit validated block to state

### run_block_proposal (18 steps)
The core block processing sequence:
1. Initialize GroveDB transaction
2. Determine epoch info and fee multipliers
3. Decode/validate state transitions
4. Execute state transitions (transform_into_action → drive operations → apply)
5. Process validator set changes
6. Handle withdrawal transactions
7. Update masternode identities and reward shares
8. Distribute fees to epoch pools
9. Commit GroveDB transaction
10. Update platform state cache

## SDK

### Fetch Pattern
```rust
let identity = Identity::fetch(&sdk, id).await?;
let docs = Document::fetch_many(&sdk, query).await?;
```
All fetches request proofs from the server and verify them locally against the platform's root hash. The `FromProof` trait handles deserialization and Merkle proof verification.

### Put Pattern
```rust
document.put_to_platform(&sdk, document_type, entropy, key).await?;
```
Internally: resolve nonce → build transition → sign → broadcast → wait for confirmation.

### BLAST Sync
Privacy-preserving identity discovery using trunk/branch tree scanning pattern. The `KeyLeafTracker` manages which identity key index ranges have been scanned. Trunk queries are broad (less private but efficient), branch queries are targeted (more private).

## WASM Bindings

### Wrapper Pattern
```rust
pub struct IdentityWasm { inner: Identity }

impl From<Identity> for IdentityWasm { ... }
impl From<IdentityWasm> for Identity { ... }

#[wasm_bindgen]
impl IdentityWasm {
    #[wasm_bindgen(js_name = getId)]
    pub fn get_id(&self) -> IdentifierWrapper { ... }
}
```

### Error Handling in WASM
The `generic_consensus_error!` macro generates wrapper structs for each consensus error variant, converting them to JsValue for JavaScript consumption. Uses the `paste!` crate for identifier concatenation.

## Testing

### TestPlatformBuilder
```rust
let platform = TestPlatformBuilder::new()
    .with_config(config)
    .with_latest_protocol_version()
    .build_with_mock_rpc()      // MockCoreRPCLike — no real Dash Core needed
    .set_genesis_state();        // write initial state tree
```

### PlatformTestConfig
Feature-gated (`#[cfg(feature = "testing-config")]`), two profiles:
- `Default::default()` — full verification (block signing, commit sigs)
- `default_minimal_verifications()` — skips crypto for speed

### Strategy Tests
Two-layer simulation framework for multi-block integration tests:
- **Strategy** — application-level: what operations to perform (document creates, identity top-ups, etc.) with `Frequency` (times_per_block_range + chance_per_block)
- **NetworkStrategy** — network-level: masternode count, quorum config, proposer changes, failure injection
- `run_chain_for_strategy(platform, block_count, strategy, config, seed)` — deterministic simulation
- `continue_chain_for_strategy` — resume from a previous outcome for restart testing

### Test Conventions
- Always use `StdRng::seed_from_u64()` for deterministic randomness
- Use `assert_matches!` for error variant checking
- Process transitions through `process_raw_state_transitions` (same code path as production)
- `OnceLock` for expensive immutable test resources (e.g., proving keys)

## System Contracts

- **DPNS** — Dash Platform Name Service (username registration, contested resources)
- **DashPay** — Social/contact features
- **Withdrawals** — Credit-to-L1 withdrawal processing
- **Masternode Reward Shares** — Reward distribution configuration
- **Feature Flags** — Protocol feature activation
- **Wallet Utils** — Wallet-related utilities
- **Token History** — Token operation audit trail
- **Keyword Search** — Searchable metadata

## Build & Development

```sh
yarn setup                              # Initial setup
cargo test --workspace                  # Run all Rust tests
cargo clippy --workspace                # Lint
cargo build -p drive-abci               # Build specific crate
cargo test -p drive-abci --lib          # Test specific crate (lib tests only)
cargo test -p drive-abci --test='*'     # Test specific crate (integration tests)
```

- **Rust edition:** 2021
- **MSRV:** 1.92
- **PR format:** Conventional Commits
- iOS development: build `packages/rs-sdk/` with `build-ios-direct.sh`

## Design Philosophy

Four core principles:
1. **Version everything** — every consensus-critical method gets a version number, looked up from PlatformVersion at runtime
2. **Explicit costs** — every operation tracks its storage and processing cost for fee calculation
3. **Clear transformation stages** — data flows through well-defined tiers (transition → action → drive op → low-level op → GroveDB)
4. **Trait-based polymorphism** — versioned enums with accessor traits, not inheritance

## Key Conventions for Contributors

- The versioned dispatch pattern is mandatory for all consensus-critical code
- Consensus errors must never have their variant order changed
- State reads happen in `transform_into_action`; state writes happen when actions are applied — never mix these phases
- `DriveHighLevelOperationConverter::into_high_level_drive_operations` must be pure (no state access)
- Fee estimation and actual execution must follow the same code paths (test both)
- Documents are not self-describing — always pair with their document type for serialization
- Use `PartialIdentity` when you only need a subset of identity fields
- Address-based transitions (v10+) use a fundamentally different authentication model than identity-based ones
