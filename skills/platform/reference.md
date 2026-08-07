# Dash Platform — On-Demand Reference

Subsystem detail loaded only when the diff touches the matching paths. The
always-loaded architectural context is in `review-core.md`; nothing here
repeats it.

Each `##` section below opens with an HTML-comment path marker listing its
globs. The orchestrator matches those globs against the lane's changed files and
injects only the matching sections. Sections marked `(manual)` are never loaded
automatically.

## SDK
<!-- paths: packages/rs-sdk/**, packages/rs-sdk-ffi/**, packages/rs-unified-sdk-ffi/**, packages/rs-unified-sdk-jni/**, packages/rs-drive-proof-verifier/**, packages/rs-context-provider/**, packages/rs-sdk-trusted-context-provider/**, packages/swift-sdk/**, packages/kotlin-sdk/**, packages/js-dash-sdk/**, packages/js-evo-sdk/** -->

`dash-sdk` is a client of Drive with the `verify` feature only — it must never
pull in `server`. `SdkBuilder` handles configuration and supports normal mode
(gRPC to the network) and mock mode (a local Drive instance for testing).

### Fetch Pattern
```rust
let identity = Identity::fetch(&sdk, id).await?;
let docs = Document::fetch_many(&sdk, query).await?;
```
All fetches request proofs from the server and verify them locally against the
platform root hash. The `FromProof` trait handles deserialization and Merkle
proof verification. A fetch path that returns data without verifying the proof
defeats the SDK's whole trust model — treat it as blocking.

### Put Pattern
```rust
document.put_to_platform(&sdk, document_type, entropy, key).await?;
```
Internally: resolve nonce → build transition → sign → broadcast → wait for
confirmation. Nonce resolution and broadcast must stay ordered; a cached or
stale nonce produces a transition the platform rejects after charging fees.

### BLAST Sync
Privacy-preserving identity discovery using a trunk/branch tree scanning
pattern. `KeyLeafTracker` manages which identity key index ranges have been
scanned. Trunk queries are broad (less private, more efficient); branch queries
are targeted (more private). Review changes here for privacy regressions —
widening a branch query into a trunk query leaks which keys a user owns.

## WASM Bindings
<!-- paths: packages/wasm-dpp/**, packages/wasm-dpp2/**, packages/wasm-sdk/**, packages/wasm-drive-verify/** -->

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

Wrappers are thin: the inner Rust type stays canonical, and the wrapper exists
only to cross the JS boundary. Logic that lives in a wrapper instead of the
inner type is invisible to the native path and will drift.

`js_name` renames are part of the public JS API — renaming one is a breaking
change for JS consumers even though Rust compiles fine.

### Error Handling in WASM
The `generic_consensus_error!` macro generates a wrapper struct per consensus
error variant, converting them to `JsValue` for JavaScript consumption. It uses
the `paste!` crate for identifier concatenation. A new consensus error variant
that is not registered here surfaces to JS as an opaque/generic error.

## Testing
<!-- paths: **/tests/**, packages/strategy-tests/**, packages/platform-test-suite/**, packages/simple-signer/** -->

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

A test asserting consensus behavior under `default_minimal_verifications()` is
not testing what it claims to test.

### Strategy Tests
Two-layer simulation framework for multi-block integration tests:
- **Strategy** — application level: which operations to perform (document
  creates, identity top-ups, …) with `Frequency`
  (`times_per_block_range` + `chance_per_block`)
- **NetworkStrategy** — network level: masternode count, quorum config,
  proposer changes, failure injection
- `run_chain_for_strategy(platform, block_count, strategy, config, seed)` —
  deterministic simulation
- `continue_chain_for_strategy` — resume from a previous outcome, used for
  restart/recovery testing

### Test Conventions
- Always use `StdRng::seed_from_u64()` for deterministic randomness. Any
  unseeded RNG in a consensus test makes failures unreproducible.
- Use `assert_matches!` for error variant checking rather than string matching
  on error `Display` output
- Process transitions through `process_raw_state_transitions` so tests exercise
  the same code path as production
- `OnceLock` for expensive immutable test resources (e.g. proving keys)

## Build & Development
<!-- paths: (manual) -->

Not loaded automatically. Pull this in only when a review question is
specifically about build tooling.

```sh
yarn setup                              # Initial setup
cargo test --workspace                  # Run all Rust tests
cargo clippy --workspace                # Lint
cargo build -p drive-abci               # Build specific crate
cargo test -p drive-abci --lib          # Test specific crate (lib tests only)
cargo test -p drive-abci --test='*'     # Test specific crate (integration tests)
```

- Rust edition 2021, MSRV 1.92 (`rust-version` in the workspace `Cargo.toml`)
- PR format: Conventional Commits
- iOS: `packages/swift-sdk/build_ios.sh` (with `setup_ios_build.sh` for
  first-time setup)
