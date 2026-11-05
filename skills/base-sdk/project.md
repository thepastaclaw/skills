# Base SDK — Project Understanding

## Overview

`dashpay/base-sdk` is a Rust workspace of `no_std` + `alloc` crates providing parsing, consensus
(de)serialization and stateless verification primitives for Dash's layer 1 chain. It is an alternate
implementation of Dash Core's wire/consensus formats; the README warns that deviations from Dash Core
(the reference implementation) are possible and crates have uneven conformance and test rigour.
There is no networking code in any crate: the SDK only encodes, decodes and verifies data.

Default branch: `develop` (CodeRabbit and CI target it). License MIT. Edition 2021, resolver 3.
MSRV: Rust `1.89.0` (`workspace.package.rust-version`). `rust-toolchain.toml` pins
`nightly-2026-02-01` for development and linting only (rustfmt uses nightly options); the repo is
validated against MSRV and `stable` (`docs/dev/getting_started.md`). Docs: https://dashpay.github.io/base-sdk.

## Crate Layout (`pkgs/`)

| Crate (dir) | Version | Responsibility |
|-------------|---------|----------------|
| `dash-types` (`types`) | 0.2.0-beta | Byte/integer newtypes, `BaseCodec` + `DecodeError`, `CompactSize`, serde helpers, `ArrayBuf` secret buffers, `impl_type!`/`make_*!` macros |
| `dash-types-marker` (`types/marker`) | 0.1.0-beta | Proc-macros `#[derive(Unencodable)]` (compile-time guard against `BaseCodec`) and `#[derive(TypeId)]` (XXH32 of type name) |
| `dash-num` (`num`) | 0.2.0-beta | `Hash256`/`Hash160`, `Arith256`, `CompactTarget` (nBits) |
| `dash-pkc` (`pkc`) | 0.2.0-beta | BLS (Chia/legacy + IETF schemes, IES, threshold, aggregation) on `blst`; ECDSA (`secp256k1`); EdDSA (`ed25519-dalek`) |
| `dash-pow` (`pow`) | 0.1.0-beta | X11 proof-of-work hash (11 chained 512-bit hashes, `digest` streaming API, `wide` SIMD) |
| `dash-script` (`script`) | 0.0.0 | Opcodes, sigop counting, address derivation (`AddrParams`) |
| `dash-primitives` (`primitives`) | 0.0.0 | Blocks, transactions, special-tx payloads (ProTx, CbTx, quorum commitment, asset lock/unlock, MnHF), governance objects/votes, `NetInfo`/`NetAddr` |
| `dash-params` (`params`) | 0.0.0 | Chain params + genesis for `Main`, `Testnet3`, `Regtest` (no devnet) |
| `dash-p2p-core` (`p2p_core`) | 0.0.0 | P2P message types, BIP324 v2 framing/short IDs only (`decode_v2`/`encode_v2`) |
| `dash-dev` (`dev`) | 0.0.0, `publish = false` | Test/bench support (JSON5 corpus reader, serde round-trip helpers) and `bsdk-util` binary (feature `bin`) |

Dependency direction (verified from Cargo.toml): `types` (<- `marker`) -> `num` -> `pkc` -> `script`;
`pow` has no internal deps; `primitives` -> {num, pkc, pow, script, types}; `params` -> {num,
primitives, script, types}; `p2p_core` -> {num, pkc, script, params, primitives, types}. `dash-dev` is a
dev-dependency of most crates. `fuzz/` is a separate workspace (`dash-pow-fuzz`, AFL++, `cargo-afl 0.18.2`).
Internal deps must specify both `path` and `version`.

## Design Invariants

- `#![no_std]` + `extern crate alloc` is mandatory; `std` is an optional feature (`AGENTS.md`).
  Standard layout: `default = []` (some crates default `codec`/`portable`), `std`, `full`, `serde`,
  `portable`. `serde` is independent of `codec` (decoupled in #60). Sealed traits are used for closed sets.
- `#![deny]`/workspace lints: `unsafe_code`, `unwrap_used`, `expect_used`, `panic`, `todo`,
  `unimplemented`, `unreachable` are all `deny`; `missing_debug_implementations` denied. Only three
  `unsafe` allowances exist: `pkc/src/bls/blst_ffi.rs` (blst C FFI), `pkc/src/ecdsa` (buffer
  zeroization) and `pow/src/aes` (intrinsics). Semgrep rule `pkc-no-unsafe-code` limits the pkc one.
- Untrusted wire input: `dash_types::codec::DecodeError` is the shared error. `CompactSize::decode`
  rejects non-minimal encodings and values above `MAX_SER_SIZE` (32 MiB = Dash Core `MAX_SIZE`);
  `decode_len`/`decode_vec(limit)` check counts before allocating; `Vec<T>` decoding reserves in
  batches capped by `MAX_VECTOR_ALLOCATE` (5,000,000). `CompactSize::decode_unbounded` exists only
  for values used as numbers (used for `ServiceFlags` in `p2p_core/src/msg/addr.rs`). Per #71 the
  caller supplies the limit when decoding from the wire.
- Other caps present: `MAX_P2P_PAYLOAD_SIZE` 3 MiB, `MAX_HEADERS` 2000 (`headers.rs`, `headers2.rs`), block size
  1,000,000 (legacy) / 2,000,000 (DIP0001), `MAX_TX_EXTRA_PAYLOAD` 10,000, `MAX_SPTX_PAYLOAD_SIZE`
  10,240, `MAX_IES_RECIPIENTS` 2048, `NetInfo` domain 253 bytes, `GovVote` sig 65/96 bytes.
- Consensus parity is the goal, with Dash Core (v24.0.0-rc.3 cited in PRs) as the oracle. Where Core is
  quirky the SDK follows Core even at the cost of round-tripping (e.g. `bool` decodes any nonzero byte as
  `true`, encodes 0/1; #71). BLS has two schemes (`BlsScChia` legacy, `BlsScIetf`); byte bags are
  scheme-less and `is_canonical::<S>()`/`to::<S>()` select scheme.
- Secrets: `zeroize`, `subtle` constant-time compare, redacting `Debug`; CodeQL `secret.ql` enforces.
- `dash-pow` and `dash-pkc` (`bls`) hit `compile_error!` on x86_64/aarch64/wasm32 unless target
  features are set (`.cargo/config.toml`) or `portable` is enabled; this is intentional.
- Source rules: 2-space indent, width 120, comment width 80, Latin-1 source, no separator comments,
  `#[expect]` with `reason =` over `#[allow]`, `From` not `Into`, `Unknown(raw)` variants carry the
  value, file header + import order per `AGENTS.md`. Test corpora are JSON5 under `pkgs/*/corpus/`.

## Key Dependencies

- rust-bitcoin crates (`base58ck`, `bitcoin_hashes`, `bitcoin-consensus-encoding`, `bitcoin-internals`,
  `bitcoin-primitives`, `bitcoin-units`) are git deps pinned to rev `7ba35c7c1dd6` (with `version` kept
  so `cargo publish` resolves from crates.io). This is the same revision `dashpay/rust-dashcore`
  uses for `bitcoin-crypto`; #72 aligned them so `cargo vendor` of the Platform workspace works.
- Crypto: `blst 0.3`, `secp256k1 0.33.1`, `ed25519-dalek 3`, `aes 0.9`, `sha2 0.11`, `digest 0.11`,
  `wide 1.7`. Tests: `rstest`, benches: `divan`, Python tooling via `uv` (`uv.lock`).
- `cargo-deny` (`.cargo/deny.toml`): advisories (yanked = deny), license allow-list, `hex-literal`
  banned (use `hex_conservative`). `rust-dashcore` is a downstream consumer (pins `dash-{num,pkc,types}`
  by git rev), not a dependency of this repo.

## Build, Test, Fuzz, CI

```bash
just test lint                    # required verification per AGENTS.md
just comb                         # runs `just test lint fuzz::test` on every commit develop..HEAD
cargo hack --workspace --exclude-features default --each-feature build
cargo hack --workspace --exclude-features default --feature-powerset clippy --all-targets -- -D warnings
cargo hack --workspace --exclude-features default --each-feature nextest run --config-file .cargo/nextest.toml
just fuzz::test                   # replay committed AFL++ corpora vs sphlib (needs `git submodule update --init`)
python3 maint/lint/lint_commit.py -m "<pr title>"
```

`just lint` runs `just --fmt --check`, `maint/lint_all.py` (rustfmt, taplo, cargo-deny, semgrep rules in
`maint/semgrep/rust`, ruff, markdown, nix, eslint) and CodeQL with custom Rust queries in
`maint/codeql/rust` (`import`, `pkc`, `attrib`, `decl`, `trait`, `secret` rules). Mutation testing is
configured (`.cargo/mutants.toml`, excludes `dev` and `pow`, `just mutants`) but no workflow runs it.

CI (`.github/workflows`): per-crate `pkg_<crate>.yml` (path-filtered) calls `build_msrv.yml` (MSRV 1.89 via
Nix devshell on ubuntu-24.04-arm: each-feature build/test, doctests, llvm-cov to Codecov with per-crate
flags, benches smoke; `pkg_pow` also replays fuzz corpora). `build_stable.yml` runs lint, PR title and
commit-message lint, and stable workspace builds on arm64 Linux, Windows and macOS. `build_cross.yml`
cross-builds x86_64 Linux/Windows and aarch64 macOS. PRs over 100 commits fail (`.github/ci.env`).
CodeRabbit (`.coderabbit.yml`) auto-reviews PRs to `develop` unless labeled `tag:skip-review`.

## PR Title and Label Conventions

Title and every commit subject: `<namespace>%<type>[(scope)][!]: <description>`, enforced by
`maint/lint/lint_commit.py` against `maint/commit.toml`. Namespaces: `num p2p_core params pkc pow
primitives script sdk types`. Types: `build chore ci doc feat fix lint misc perf refac test`.
`!` marks a breaking change. Commit messages max 73 columns wide, 5 lines tall. AI tool co-author
trailers/authors (Claude, Codex, Copilot, etc.) are blocked by lint. Examples: `p2p_core%feat!: ...`,
`sdk%ci: ...`, `pow%test: ...`.

Labels: `topic:Build/CI`, `topic:Codec`, `topic:Crypto`, `topic:Docs`, `topic:P2P`, `topic:Primitives`
(and `topic:Bazel/FFI`, configured in `.github/board.json` but no Bazel files exist in the repo);
`tag:skip-review`, `tag:stale` (60 days), `tag:needs-rebase`. Per PR #60, repository policy prohibits
squash merges; `contrib/git_merge.py` bakes the PR description (with agent disclosure) into the merge
commit. The PR template asks for Motivation, Additional Information, Breaking Changes, How Has This
Been Tested (commands), a checklist and an agent disclosure.
Breaking changes for published crates go in `pkgs/<crate>/CHANGELOG.md` (Keep a Changelog); unpublished
crates (`primitives`, `p2p_core`, `params`, `script`) have stub changelogs ("Initial release") and
describe breaking changes in the PR body.
