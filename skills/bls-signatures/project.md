# BLS Signatures — Project Understanding

## Overview

`dashpay/bls-signatures` is a fork of Chia Network's BLS signature library implementing BLS12-381 signatures with aggregation. It provides the cryptographic foundation for Dash's threshold signing, LLMQ quorum operations, InstantSend, and ChainLocks. Dash maintains its own fork with Dash-specific additions (legacy scheme, threshold signing, mimalloc integration, Rust/Go/JS/Python bindings).

## Source Directory Structure

```
.
├── include/dashbls/       # Public C++ headers
│   ├── bls.hpp            # Core BLS namespace, initialization
│   ├── elements.hpp       # G1Element, G2Element (public keys, signatures)
│   ├── privatekey.hpp     # PrivateKey class
│   ├── schemes.hpp        # CoreMPL, BasicSchemeMPL, AugSchemeMPL, PopSchemeMPL, LegacySchemeMPL
│   ├── threshold.hpp      # Threshold signing (Shamir's secret sharing)
│   ├── legacy.hpp         # Legacy (pre-V19) BLS scheme
│   ├── hdkeys.hpp         # HD key derivation (EIP-2333)
│   ├── hkdf.hpp           # HKDF key derivation
│   ├── chaincode.hpp      # BIP-32-like chain codes
│   ├── extendedprivatekey.hpp
│   ├── extendedpublickey.hpp
│   └── util.hpp           # Utility functions
├── src/                   # C++ implementation
│   ├── bls.cpp            # BLS init, cleanup, scheme flag
│   ├── elements.cpp       # G1/G2 element operations
│   ├── privatekey.cpp     # Key generation, signing
│   ├── schemes.cpp        # Signing schemes implementation
│   ├── threshold.cpp      # Threshold signing
│   ├── legacy.cpp         # Legacy scheme
│   ├── test.cpp           # C++ test suite (Catch2)
│   └── test-bench.cpp     # Benchmarks
├── rust-bindings/         # Rust FFI bindings
│   ├── bls-signatures/    # Safe Rust wrapper crate
│   │   └── src/           # Rust API (elements, keys, schemes)
│   └── bls-dash-sys/      # Raw C bindings (bindgen)
│       ├── c-bindings/    # C API headers for FFI
│       └── build.rs       # Build script (compiles C++ lib)
├── go-bindings/           # Go CGo bindings
├── js-bindings/           # JavaScript/WASM bindings
├── python-bindings/       # Python (pybind11) bindings
├── python-impl/           # Pure Python reference implementation
├── depends/
│   └── relic/             # Relic toolkit (pairing-based crypto primitives)
│   └── mimalloc/          # Memory allocator
│   └── catch2/            # Test framework
├── CMakeLists.txt         # CMake build (primary)
├── Makefile.am            # Autotools build (for Dash Core integration)
└── apple.rust.deps.sh     # Apple platform Rust dependency script
```

## Cryptographic Foundation

### BLS12-381 Curve
- Pairing-friendly elliptic curve
- G1: 48-byte public keys (compressed)
- G2: 96-byte signatures (compressed)
- Supports efficient signature aggregation
- Implemented via the **relic toolkit** (`depends/relic/`)

### Signing Schemes (IETF BLS RFC)
- **BasicSchemeMPL**: Minimal-pubkey-size, basic signing
- **AugSchemeMPL**: Augmented (domain separation with pubkey prepended to message)
- **PopSchemeMPL**: Proof of Possession (requires PoP before aggregation)
- **LegacySchemeMPL**: Dash-specific legacy scheme (pre-V19, non-standard)

### Threshold Signing
- Shamir's Secret Sharing for key splitting
- `Create()` — split private key into shares
- `SignWithCoefficient()` — sign with Lagrange coefficient
- `AggregateUnitSigs()` — combine threshold signatures
- Used by Dash LLMQ quorums for distributed signing

### HD Key Derivation
- EIP-2333 key derivation
- BIP-32-like unhardened key derivation
- Extended private/public key support

## Dash-Specific Additions

### Legacy Scheme
- `bls::bls_legacy_scheme` global flag controls active scheme
- Legacy scheme uses different domain separation
- Dash Core V19 deployment switched from legacy to basic scheme
- Both schemes must remain functional for historical verification

### Mimalloc Integration
- `depends/mimalloc/` — custom memory allocator
- `Makefile.mimalloc.include` — build integration
- Used for performance in memory-intensive BLS operations

## Build Systems

### CMake (primary, for standalone builds)
```bash
mkdir build && cd build
cmake .. -DBUILD_BLS_TESTS=ON
cmake --build . -- -j$(nproc)
./src/runtest  # Run tests
```

### Autotools (for Dash Core integration)
```bash
./autogen.sh
./configure
make
```

### Rust Bindings
```bash
cd rust-bindings/bls-signatures
cargo build
cargo test
```

## Testing

- **C++ tests:** `src/test.cpp` using Catch2 framework
  - Tests for all schemes (Basic, Aug, Pop, Legacy)
  - Aggregation tests
  - Threshold signing tests
  - HD key derivation tests
  - Serialization roundtrip tests
- **Rust tests:** `cargo test` in `rust-bindings/bls-signatures/`
- **Python tests:** `python-bindings/test.py`
- **Benchmarks:** `src/test-bench.cpp`

## Key Types

- `G1Element` — point on G1 (48 bytes compressed) — public keys
- `G2Element` — point on G2 (96 bytes compressed) — signatures
- `PrivateKey` — BLS private key (32 bytes)
- `ExtendedPrivateKey` / `ExtendedPublicKey` — HD keys with chain codes
- `Threshold` — namespace for threshold signing operations

## Upstream Relationship (Chia Network)

- Forked from `Chia-Network/bls-signatures`
- Dash additions: legacy scheme, threshold signing enhancements, Dash-specific bindings
- Upstream changes may be cherry-picked but require careful testing against Dash-specific code

## Key Maintainers

- **PastaPastaPasta** — Primary maintainer
- Used as a dependency by Dash Core and Dash Platform (via Rust bindings)
