# BLS Signatures — Review Skill

## Code Style

- C++14/17 (not C++20 — older than Dash Core)
- `clang-format` enforced (`.clang-format` in repo root)
- Headers in `include/dashbls/` — public API
- Implementation in `src/` — private
- Naming: `CamelCase` for types, `camelCase` for methods, `UPPER_CASE` for constants
- Rust bindings follow standard Rust conventions (`snake_case`)

## Security-Critical Areas

**Every change in this repo is potentially security-critical.** BLS signatures are consensus-critical infrastructure for:
- LLMQ threshold signing (InstantSend, ChainLocks)
- Masternode authentication
- Platform validator consensus

### Extra Scrutiny Required

1. **Relic toolkit integration** — any change to how relic primitives are called
2. **Serialization/deserialization** — malformed input handling, buffer overflows
3. **Scheme selection** — legacy vs basic scheme flag handling
4. **Threshold operations** — Lagrange coefficient computation, share validation
5. **Memory handling** — relic uses its own memory model, mixing with C++ allocation is risky
6. **Constant-time operations** — BLS private key operations must not leak timing info
7. **Input validation** — point-on-curve checks, subgroup checks, infinity point handling

## Common Pitfalls

- **Legacy/Basic scheme confusion**: Any code that touches signing or verification must be aware of `bls::bls_legacy_scheme`. Tests should cover both schemes.
- **Relic initialization**: `BLS::Init()` must be called before any operations. Thread safety of relic is limited.
- **G1/G2 point validation**: Deserialized points must be validated (on-curve, in subgroup). Skipping validation is a security vulnerability.
- **Aggregation with empty inputs**: Edge case — aggregating zero signatures/keys must be handled safely.
- **Memory leaks in C bindings**: The C API (`c-bindings/`) allocates memory that callers must free. Missing free = leak. Double free = vulnerability.
- **Rust FFI safety**: `bls-dash-sys` uses raw pointers. The safe wrapper in `bls-signatures` must correctly manage lifetimes.
- **Cross-platform build differences**: macOS, Linux, and Windows may have different relic configurations.

## Test Expectations

- **All scheme changes**: Must test with both legacy and basic schemes
- **New operations**: Must have Catch2 tests in `src/test.cpp`
- **Rust binding changes**: Must have corresponding Rust tests
- **Serialization changes**: Must include roundtrip tests (serialize → deserialize → compare)
- **Threshold changes**: Must test with various threshold/total configurations
- **Edge cases**: Empty inputs, invalid inputs, max-size inputs

## Things NOT to Flag

- Relic toolkit internal code style (it's a dependency, not our code)
- Python reference implementation style (`python-impl/` is reference, not production)
- Minor differences between binding languages (Go/JS/Python may lag behind C++)
- `TODO` comments in upstream Chia code
- mimalloc configuration quirks (it works, don't touch it)

## Consensus-Critical Code Areas

**All of these are consensus-critical:**
- `src/schemes.cpp` — signing and verification
- `src/threshold.cpp` — threshold operations
- `src/elements.cpp` — point serialization/deserialization
- `src/privatekey.cpp` — key generation
- `src/legacy.cpp` — legacy scheme implementation
- `include/dashbls/*.hpp` — public API contracts

Changes here should have:
1. Clear reasoning in PR description
2. Comprehensive tests covering both BLS schemes
3. Consideration of backwards compatibility (serialized data on-chain)
4. No breaking changes to the C/Rust/Go API without version bump

## Build System Notes

- CMake is the primary build system for standalone development
- Autotools (`Makefile.am`, `Makefile.*.include`) is used when building as part of Dash Core
- Both must be kept in sync when adding/removing source files
- Rust bindings have their own `build.rs` that compiles the C++ library

## Binding Review Checklist

When reviewing changes to language bindings:
1. Does the C binding correctly handle memory ownership?
2. Does the safe wrapper (Rust/Go/Python) prevent use-after-free?
3. Are error codes propagated correctly across the FFI boundary?
4. Is the binding tested independently (not just via C++ tests)?
5. Does `include.h` / `c-bindings/` expose the right subset of the API?
