# Dash Platform — Core Review Context

Always-loaded architectural context for reviewing `dashpay/platform`: the
invariants a reviewer needs on *every* diff — dependency direction, versioned
dispatch, the state-transition validation pipeline, the operation pipeline, the
data model, fees, errors, serialization, ABCI. Subsystem detail that only
matters when the diff touches it (SDK, WASM, testing harnesses, build commands)
lives in `reference.md` and is loaded on demand.

Facts checked against `origin/v4.2-dev` (default branch), August 2026. Counts
carry an as-of marker; when a count matters to a finding, re-read the cited
source rather than trusting this document.

## Architecture & Core Dependency Chain

A monorepo of ~47 workspace Rust crates plus JS/TS packages: identities, data
contracts, documents, tokens, gRPC API. Runs on masternodes via Tenderdash (a
CometBFT fork); state lives in GroveDB, a Merkle-authenticated database on
RocksDB.

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
| **Client** | `dash-sdk`, `wasm-dpp` / `wasm-sdk` (WASM), `rs-sdk-ffi` / `rs-unified-sdk-*` (mobile FFI), JS SDK packages |
| **Consensus** | `rs-tenderdash-abci` (ABCI interface), signature/key crates |
| **Testing** | `strategy-tests`, `simple-signer`, test helpers |

- **DPP** is storage-agnostic and shared by client and platform; it carries
  dozens of feature flags (~77 as of Aug 2026), and new code must respect the
  existing gates.
- **Drive** splits on features: `server` (full node — reads, writes, proof
  generation) vs `verify` (proof verification only, used by the SDK). Drive owns
  the `state_transition_action` types: the validated, resolved representations
  applied to storage.
- **Drive-ABCI** is organized into `abci/` (Tenderdash interface), `execution/`
  (block and transition processing), `query/` (gRPC handlers).

## Versioning System

`PlatformVersion` is an immutable snapshot pinning every versioned behavior,
nested several levels deep (`drive_abci.validation_and_processing
.state_transitions.*`, `drive.methods.*`, …). `FeatureVersion` is a `u16`;
`OptionalFeatureVersion` is `Option<u16>` for features introduced later. The
structs sit in a `PLATFORM_VERSIONS` array indexed by protocol version, looked
up with `PlatformVersion::get(protocol_version)`. `LATEST_VERSION` in
`packages/rs-platform-version/src/version/mod.rs` is authoritative (v14 as of
Aug 2026) — never hardcode a version count in a finding.

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

**Directory convention:** `mod.rs` (dispatcher) + `v0/mod.rs`, `v1/mod.rs`.

**Adding a version:** create `v{N}/mod.rs` → add the match arm in `mod.rs` → add
the version number to the appropriate `PlatformVersion` structs → set it in the
latest platform version constant.

Editing an existing `v{N}` implementation in place changes behavior for a
protocol version that is already live. That is a consensus break, not a fix.

## State Transitions

`packages/rs-dpp/src/state_transition/mod.rs` — 21 variants as of Aug 2026:

```rust
pub enum StateTransition {
    DataContractCreate, DataContractUpdate,
    Batch,                                   // documents + tokens
    IdentityCreate, IdentityTopUp, IdentityCreditWithdrawal,
    IdentityUpdate, IdentityCreditTransfer,
    MasternodeVote,
    // address-based (v11+)
    IdentityCreditTransferToAddresses, IdentityCreateFromAddresses,
    IdentityTopUpFromAddresses, AddressFundsTransfer,
    AddressFundingFromAssetLock, AddressCreditWithdrawal,
    // shielded pool (v12+)
    Shield, ShieldedTransfer, Unshield, ShieldFromAssetLock,
    ShieldedWithdrawal, IdentityCreateFromShieldedPool,
}
```

The enum is `#[platform_serialize(unversioned)]` and serde-tagged with `$type`.
`BatchTransition` aggregates multiple document/token operations into one
transition. Address-based variants use UTXO-style inputs/outputs instead of
identity-key signing; shielded variants authenticate with zero-knowledge proofs.
**Inserting or reordering variants anywhere but the end changes the binary wire
encoding — treat it as consensus-breaking.**

### State-Transition Validation Pipeline

Distinct from the *ABCI request flow* (check_tx / prepare_proposal /
process_proposal / finalize_block, below) — this is the per-transition stage
sequence run inside those handlers. Authoritative ordering:
`packages/rs-drive-abci/src/execution/validation/state_transition/processor/v0/mod.rs`,
one trait per stage under `processor/traits/`. In order:

1. **is_allowed** — is this transition type enabled at the current protocol version?
2. **identity-based signature verification**
3. **address witness validation** — address-based transitions
4. **address balances & nonces** — address-based inputs have sufficient funds
5. **identity nonces** — replay protection for identity-based transitions
6. **basic structure** — schema-level validation (types, required fields, sizes)
7. **minimum balance pre-checks** — identity balance, address balances, and
   prefunded specialized balance, depending on transition type
8. **shielded checks** — minimum shielded fee, then shielded proof verification
9. **advanced structure** — deeper stateless structural validation
10. **transform_into_action** — resolve references against state, produce a
    `StateTransitionAction`
11. **advanced structure from state** — structural checks needing the action
12. **state validation** — final checks against current platform state

A transition (what the client sends) becomes an action (what the platform
executes). Actions live in `rs-drive`, transitions in `rs-dpp`. Reordering or
skipping a stage is a consensus violation; a new transition type must implement
every stage trait that applies to it.

### System Actions

When a transition fails validation *after* the nonce check, the platform still
must bump the nonce and charge fees.
`packages/rs-drive/src/state_transition_action/system/`:
`BumpIdentityNonceAction`, `BumpIdentityDataContractNonceAction`,
`PartiallyUseAssetLockAction`, `BumpAddressInputNoncesAction`.

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

- A single document create can produce 10+ GroveDB operations (one per index,
  plus document and metadata)
- `into_high_level_drive_operations` is pure — no state reads or writes
- `into_low_level_drive_operations` may read state for cost estimation
- With `apply = false` the pipeline runs in estimation mode for fee pre-checks;
  estimation and execution must traverse the same code paths
- `LowLevelDriveOperation` carries more than tree mutations: alongside
  `GroveOperation` there are pure-cost variants (`FunctionOperation` for CPU work
  such as hashing and signature verification, `CalculatedCostOperation`,
  `PreCalculatedFeeResult`). New CPU-heavy work that pushes no `FunctionOperation`
  is undercharged, and undercharging is consensus-visible.

## Data Model

**Data contracts** — versioned enum (`DataContract::V0/V1`) with accessor traits
(`DataContractV0Getters`: `id()`, `owner_id()`, `document_types()`). V1 adds
groups (multi-sig authorization), tokens, timestamps, associated token events,
and keywords. Contracts define document types with JSON Schema validation, index
rules, and storage config. Contract ID is derived deterministically from owner ID
+ identity nonce.

**Documents** — **not self-describing**: deserialization requires the document
type definition from the contract. ID is
`sha256(contract_id + owner_id + document_type_name + entropy)`. Fields: `id`,
`owner_id`, `revision`, `created_at`, `updated_at`, `transferred_at`,
`properties` (BTreeMap). Accessors: `DocumentV0Getters` / `DocumentV0Setters`.

**Identities** — `id`, `public_keys` (BTreeMap<KeyID, IdentityPublicKey>),
`balance` (Credits), `revision`. 1 Dash = 100,000,000,000 credits.
Key `purpose`: Authentication, Encryption, Decryption, Transfer, System, Voting,
Owner. `security_level`: Master, Critical, High, Medium. `key_type`:
ECDSA_SECP256K1, BLS12_381, ECDSA_HASH160, BIP13_SCRIPT_HASH,
EDDSA_25519_HASH160. Prefer `PartialIdentity` when an operation needs only some
fields. Nonces are a 40-bit monotonic counter plus a 24-bit missing-revisions
bitfield for replay protection (`IDENTITY_NONCE_VALUE_FILTER`,
`MISSING_IDENTITY_REVISIONS_FILTER`).

## Drive & GroveDB

GroveDB is a hierarchical authenticated Merkle tree database on RocksDB.
Subtrees nest arbitrarily; paths are vectors of byte-string segments. Sum trees
track aggregates (used for balance accounting). Fully transactional — all reads
and writes in a block happen in one transaction — with cost tracking built in.

Drive's pattern: a `drive_operations: &mut Vec<LowLevelDriveOperation>`
accumulator threaded through call chains.
`DirectQueryType::StatelessDirectQuery` uses estimated layer sizes instead of
reading GroveDB; `StatefulDirectQuery` reads actual state — picking the wrong one
silently diverges estimated from real cost. Post-commit callbacks (e.g. cache
invalidation) run as `DriveOperationFinalizeTask`.

## Fee System

| Era | Versions | Authentication | Fee Model |
|-----|----------|---------------|-----------|
| Identity Credits | v1–v10 | Identity keys sign transitions | Deduct credits from identity balance |
| Platform Addresses | v11+ | UTXO-style inputs/outputs | Deduct from address inputs |
| Shielded | v12+ | Zero-knowledge proofs | Three-component: proof + per-action processing + per-action storage |

Activation constants live in
`packages/rs-platform-version/src/version/feature_initial_protocol_versions.rs`
(`ADDRESS_FUNDS_INITIAL_PROTOCOL_VERSION = 11`,
`SHIELDED_POOL_INITIAL_PROTOCOL_VERSION = 12`). Eras coexist: v12+ nodes must
still execute v1-era identity-credit transitions correctly.

- **Storage fee** — proportional to bytes written. Ongoing cost (state occupies
  tree space indefinitely); partially refundable when data is deleted.
- **Processing fee** — CPU work: hashing, signature verification, tree
  traversal. Ephemeral, not refundable.
- `user_fee_increase` — percentage multiplier on processing fee for priority.
- Distribution: fees go into epoch-based pools. `PERPETUAL_STORAGE_ERAS = 50`,
  `DEFAULT_EPOCHS_PER_ERA = 40` → ~2000 distribution periods. The proposer gets
  100% of processing fees; storage fees are distributed over time.

## Error Handling

### Consensus Errors (user-facing, cross-node)
- Must be deterministic — all nodes must agree on which error occurred
- Serializable with `PlatformSerialize` / `PlatformDeserialize`
- **DO NOT CHANGE THE ORDER** of enum variants — they are serialized by index.
  Applies to `ConsensusError` and to each category enum. Append only.
- Four categories with code ranges in
  `packages/rs-dpp/src/errors/consensus/codes.rs`: `BasicError` (10000-range,
  structural/schema), `SignatureError` (20000-range, authentication),
  `FeeError` (30000-range, balance/fee), `StateError` (40000-range, state
  conflicts). New errors take a new unused code — reusing or shifting a code
  changes what clients see for an existing failure.

### Drive Errors (internal, single-node)
Not serialized, not consensus-critical. 14 variants as of Aug 2026, by
subsystem (`Query`, `StorageFlags`, `Drive`, `Proof`, `GroveDB`, `Protocol`,
`Identity`, `Fee`, `Document`, `Value`, `DataContract`, `Cache`, plus
info-string variants). Large error types are wrapped in `Box<>` to keep the enum
small. Severity spectrum: `CorruptedCodeExecution` (bug),
`CorruptedElementType` (data corruption), `NotFound` (missing data).

## Serialization

Platform serialization wraps bincode with `PlatformVersion`-aware encode/decode
(`PlatformVersionEncode` / `PlatformVersionedDecode`). Derive macros:
`PlatformSerialize`, `PlatformDeserialize`, `PlatformSignable`. Attributes:

- `#[platform_serialize(limit = 100000)]` — size cap
- `#[platform_serialize(unversioned)]` — skip version prefix
- `#[platform_serialize(into = "OtherType")]` — convert before serializing
- `#[platform_signable(exclude)]` — omit field from the signature hash

All consensus-critical types must use platform serialization, not raw
bincode/serde. The version prefix enables forward-compatible deserialization.
`PlatformSignable` produces the exact bytes that get signed; `exclude` fields
(like the signature itself) are replaced with defaults — adding a signed field
without excluding it changes every existing signature's preimage.

## ABCI Request Flow

Four Tenderdash entry points (`packages/rs-drive-abci/src/abci/`, block
execution under `execution/engine/`):

1. **check_tx** — mempool validation (lightweight, no state changes)
2. **prepare_proposal** — proposer builds block content
3. **process_proposal** — validators verify the proposed block
4. **finalize_block** — commit the validated block to state

`run_block_proposal` (`execution/engine/run_block_proposal/`) is the core block
sequence shared by prepare/process: open the GroveDB transaction, resolve epoch
info and fee multipliers, decode and validate transitions, execute them through
the three-tier pipeline, apply validator-set and masternode identity changes,
process withdrawals, distribute fees to epoch pools, commit, refresh the platform
state cache. Anything that makes prepare_proposal and process_proposal disagree
is a chain halt.

## System Contracts

`SystemDataContract` (`packages/data-contracts/src/lib.rs`) — discriminants are
stable and must not be renumbered: `Withdrawals` (0, credit-to-L1 withdrawals),
`MasternodeRewards` (1), `FeatureFlags` (2 — **reserved slot**: never deployed
at genesis, implementation removed, discriminant kept only to preserve
numbering), `DPNS` (3, name service and contested resources), `Dashpay` (4),
`WalletUtils` (5), `TokenHistory` (6), `KeywordSearch` (7),
`DocumentHistory` (8).

System contract changes need a version bump, a migration path for existing data,
and consideration of upgrade ordering.

## Key Conventions for Contributors

- The versioned dispatch pattern is mandatory for all consensus-critical code
- Consensus errors must never have their variant order changed
- State reads happen in `transform_into_action`; state writes happen when
  actions are applied — never mix these phases
- `DriveHighLevelOperationConverter::into_high_level_drive_operations` must be
  pure (no state access)
- Fee estimation and actual execution must follow the same code paths (test both)
- Documents are not self-describing — always pair with their document type for
  serialization
- Use `PartialIdentity` when you only need a subset of identity fields
- Address-based transitions (v11+) and shielded transitions (v12+) use
  fundamentally different authentication models than identity-based ones
- Rust edition 2021, MSRV 1.92; PRs use Conventional Commits
