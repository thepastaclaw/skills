# Base SDK — Review Skill

## Review Posture

- Review base-to-head against the exact head SHA. This is a consensus-parity library: for any change that
  decodes, encodes, hashes, signs or verifies, find the Dash Core code it must match and compare, instead
  of judging the Rust in isolation. Maintainers (kwvg) cite Core by permalink to specific lines
  (v24.0.0-rc.3 or a pinned commit); expect the PR description to carry those links and check them.
- A PR's stated scope matters. kwvg rejected "use a tighter protocol limit" suggestions on a PR whose
  goal was switching semantics while preserving behaviour (#71). Put out-of-scope hardening in
  `out_of_scope_findings` unless the PR makes the surface worse.
- The README states crates have uneven conformance. Do not assume a crate is complete; judge each
  change by whether it moves the crate closer to or further from Core, and whether it says so.
- The PR template requires an agent disclosure and author ownership of the code. Prefer few, verified
  findings with a Core source reference over a long list of style notes.

## Validation Expectations

Repo gate is `just test lint` (`AGENTS.md`); the individual pieces a reviewer can run are:

```bash
cargo hack --workspace --exclude-features default --each-feature build
cargo hack --workspace --exclude-features default --feature-powerset clippy --all-targets -- -D warnings
cargo hack --workspace --exclude-features default --each-feature nextest run --config-file .cargo/nextest.toml
cargo hack --workspace --exclude-features default --each-feature test --doc
RUSTDOCFLAGS="-D warnings" cargo doc --workspace --no-deps --features full
cargo fmt --check --all
python3 maint/lint/lint_commit.py -m "<PR title>"      # also -r base..head for commit subjects
just fuzz::test                                         # dash-pow only; needs the fuzz/pow/sphlib submodule
```

MSRV is validated per crate at Rust 1.89 and stable on arm64 Linux, Windows and macOS; a change that
needs a newer language or std feature is a finding. Semgrep (`maint/semgrep/rust`) and CodeQL
(`maint/codeql/rust`) encode many style and secret-handling rules; do not duplicate their output, but
do flag PRs that disable, bypass (`nosemgrep`) or weaken them. Tests are `rstest` cases; vectors live
in `pkgs/*/corpus/*.json5`. New wire types, payloads and crypto operations should add corpus vectors
sourced from Core (not derived from the SDK's own encoder) plus decode-error cases.

## Correctness Focus

- **Consensus parity with Dash Core:** field order, widths, endianness, version-dependent fields,
  conditional payload members, and accept/reject behaviour must match Core's serializer and validation.
  Where Core is lenient or odd, the SDK follows Core (e.g. `bool` decodes any nonzero byte as true,
  accepting non-round-trippable input; #71). Flag "cleanups" that make decoding stricter or more lenient
  than Core without saying so. Intentional deviations (SDK stricter than Core) need a stated reason.
- **Decoding untrusted bytes:** every length/count prefix must be range-checked before allocation or
  loops: `CompactSize::decode` (<= `MAX_SER_SIZE`), `decode_len(limit)`, `decode_vec(limit)`,
  `into_len(MAX_*)`. `CompactSize::decode_unbounded` is only for values used as numbers (service
  flags); flag it elsewhere. Look for `Vec::with_capacity(n)` from wire values, loops bounded only by
  `MAX_SER_SIZE` that allocate per element, missing trailing-byte checks (`DecodeError::TrailingBytes`),
  and silent truncation on `as` casts of decoded integers. Per-message caps live in
  `p2p_core/src/codec.rs` (`MAX_P2P_PAYLOAD_SIZE`), `headers*.rs`, `primitives/src/{transaction,block,codec}.rs`.
- **Canonical encodings / round-trips:** non-minimal CompactSize is rejected; BLS byte bags are
  scheme-less and `is_canonical::<S>()` / `to::<S>()` decide validity per scheme (legacy `BlsScChia`
  vs `BlsScIetf`); `Unknown(raw)` enum variants must preserve the raw value. Every codec change should
  keep `decode(encode(x)) == x` where Core guarantees it, and the corpus must prove it.
- **Crypto (`dash-pkc`, `dash-pow`):** BLS on `blst`: subgroup/on-curve and infinity handling, legacy
  vs IETF domain separation, aggregation with empty or duplicate inputs (`BlsError::EmptyAggregation`,
  `DuplicateMessage`, `DuplicateShareId`), threshold/share recovery, IES recipient bounds
  (`MAX_IES_RECIPIENTS`). Legacy BLS signs the SHA-256 digest of arbitrary-length messages;
  `*_digest` vs `*_message` APIs must not be confused. ECDSA verify normalizes S; callers wanting
  non-malleability use `is_low_s`. `dash-pow` X11 must match Core and `sphlib` for every stage;
  a change to any stage, the `digest` streaming API or SIMD/AES backends needs corpus plus
  `just fuzz::test`, and the `portable` path must produce identical output.
- **Params and script:** `dash-params` constants (genesis, `ConsensusParams`, address prefixes) are
  consensus values; verify against Core's `chainparams.cpp` for any edit. Sigop counting
  (`script/src/sigops.rs`) and address derivation must match Core's script rules.
- **P2P (`p2p_core`):** BIP324 v2 transport only. Short-ID tables, 3 MiB payload cap and message
  bounds should derive from tables/constants, not hand-maintained numbers. v1-transport handling is
  unwanted.
- **Breaking changes and semver:** `!` in the title for any public API/behaviour break, and the
  break must appear in the crate's `CHANGELOG.md` if the crate is published (`types`, `num`, `pkc`,
  `pow`, the four with changelog links in the README). Unpublished crates (`primitives`, `p2p_core`,
  `params`, `script`) take breaking-change text in the PR body ("Breaking Changes" section).
  `rust-dashcore` consumes `dash-num`, `dash-pkc`, `dash-types` by git rev, so a silent API change
  there breaks a downstream workspace.
- **Features:** `no_std` + `alloc` must keep building; `std`, `serde`, `codec`, `portable`, `full`
  are checked individually and as a powerset. `serde` must not imply `codec`. `full` must not include
  `portable`. New optional deps need a `dep:` entry, a feature, and `docs.rs` metadata consistency.

## Security / Trust Boundaries

- Everything decoded from the wire (P2P messages, blocks, transactions, special-tx payloads, BLS bytes,
  governance objects) is adversarial input. A panic, unbounded allocation or quadratic loop reachable
  from `decode` is a security finding. Workspace lints deny `unwrap`/`expect`/`panic`/`todo`/
  `unreachable`/`unsafe_code`; new `#[expect(unsafe_code)]` outside `bls/blst_ffi.rs`,
  `ecdsa` zeroization and `pow/src/aes` (intrinsics) is a red flag. Unsafe blocks in `pow` need
  `// SAFETY:` comments (`clippy::undocumented_unsafe_blocks` is denied there).
- Secrets (`BlsSecretKey`, `BlsSkShare`, ECDSA/EdDSA secret keys, `ArrayBuf`): must zeroize on drop,
  have redacting `Debug`, use `subtle` for equality, and not be placed in growable buffers or logged.
  Secret-typed values must not pick up `Clone`/`Display`/serde impls without justification.
- This SDK does no I/O, so network, filesystem, timing-of-RPC concerns are out of scope. Constant-time
  behaviour matters only for secret-dependent code paths (key operations, equality).
- The `blst` crate and rust-bitcoin git pin are supply-chain inputs: dependency bumps should keep
  `cargo-deny` green, preserve `Cargo.lock` consistency, and (for rust-bitcoin) stay aligned with the
  revision `rust-dashcore` uses (#72) so downstream `cargo vendor` works.

## Rust Quality Notes

- Follow `AGENTS.md` / `docs/dev/guide_rust.md`: newtypes over primitives, enums over bools, `From`/
  `TryFrom` (never `impl Into`), `as_`/`to_`/`into_` naming, getters without `get_`, acronyms as
  words, `# Errors` docs, lowercase error messages without trailing punctuation, `#[expect]` with
  `reason`. Prefer `thiserror` enums per domain.
- Low-level helpers such as `dash_types::codec::take` are public only for macros; kwvg wants decoding
  through typed `BaseCodec` impls (`u8::decode`, `CommandString::decode`) or a one-line justification.
- Newtypes belong in the owning crate, not `dash-types` (semgrep `types-macro-no-codec`). Use the
  `codec_type!`/`impl_type!` macro family rather than hand-rolled `extend_from_slice` (semgrep
  `codec-no-raw-extend`). Keep count and byte-size constants distinguishable (kwvg asked for `*_SIZE`
  on byte sizes and `*_COUNT` on counts in #67), and derive table lengths instead of hand-tracking them.
- Tests should assert behaviour, not restate constants or the implementation. Standalone/integration
  tests belong in `pkgs/<crate>/tests`; benches use `divan`; fixtures are JSON5.

## Things Not To Flag

- Formatting, import order, 2-space indentation, 120/80 column widths, Latin-1-only source: rustfmt
  (nightly options) and semgrep own these. `rustfmt.toml` using unstable options is intended.
- `compile_error!` on missing CPU target features in `dash-pow`/`dash-pkc` (`bls`), the x86-64-v3 /
  ADX `.cargo/config.toml` flags, and the `portable` escape hatch: documented system requirements.
- Pinned `nightly-2026-02-01` in `rust-toolchain.toml` (dev/lint only), the rust-bitcoin git `rev`
  pin with retained `version` (needed for `cargo publish` and the Platform vendor build), and large
  `Cargo.lock`/`uv.lock` diffs.
- Decoders that accept non-round-trippable input because Dash Core does (e.g. nonzero `bool`), and
  SDK-side caps stricter than Core's deserializer (`MAX_IES_RECIPIENTS`: kwvg noted Core has no such
  bound) when the PR is not touching them. Do not demand tighter limits than the existing
  `MAX_SER_SIZE` convention on a behaviour-preserving PR (#71); CodeRabbit asked for tighter
  `NetInfoV2`/IES counts there and kwvg answered out of scope / already bounded.
- Missing Core C++ cross-references in code comments: kwvg asked for them to be removed because they
  go stale; Core references belong in the PR description as line-anchored permalinks. Likewise do
  not ask for comments stating current limit values.
- Changelog entries for unpublished crates (stub "Initial release"), and bot noise from
  CodeRabbit, the PR-status bot ("may have conflicts"), or `tag:*` labels.
- `contrib/` scripts (dev tooling) lacking `maint/`-level defensive checks (kwvg, #70: `contrib` takes
  fewer belt-and-suspenders checks than `maint`), `fuzz/` harness style, `docs/samples`, and Nix/CI
  plumbing outside the changed surface.
- Long PR descriptions or a missing `topic:` label are maintainer-process matters; only mention an
  absent "How Has This Been Tested?" command list or absent Breaking Changes text on `!` PRs.
