# Dash Wallet Desktop — Review Skill

## Review Posture

- Review only the PR under consideration. Keep findings scoped to behavior introduced or exposed by the PR.
- This is a public repo. GitHub comments are public; avoid private operational context, hostnames and IPs.
- This wallet holds users' funds and has had no security review. Weight funds safety, key handling, data loss, wrong-network and wrong-amount bugs above everything else, then correctness of Platform/DashPay flows, then UX. Style is last.

## Expected Local Gates

From `rust/`:

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets --locked -- -D warnings
cargo test --workspace --locked --no-fail-fast
```

From `apps/desktop/`:

```bash
pnpm install --frozen-lockfile
pnpm tokens:check
pnpm typecheck
pnpm test
pnpm build && pnpm build:fixture
```

If a PR changes `dw-app` commands or types, the generated bindings must be regenerated (`DW_UPDATE_BINDINGS=1 cargo test -p dw-app`) and `apps/desktop/src/api/generated` plus `docs/design/E0-13-facade-coverage.md` committed; CI fails on a diff. Swift jobs (`swift build`, `swift test`, `scripts/build-core.sh --check-bindings`) run on macOS and Linux CI only.

Never launch the GUI app, XCUITests, Simulator or anything that raises Keychain/Touch ID prompts on the review host (`AGENTS.md`, "No visible UI").

## Wallet Safety Rules

- Amounts: duffs are integers. No floating-point arithmetic on amounts, in Rust or TypeScript; renderer values cross IPC as strings or integers that cannot lose precision. Check fee, change and "send max" math, dust limits and rounding in `dw-units`.
- Network: addresses, URIs, PSBTs, imported wallets and deep links must be checked against the active network. A testnet address accepted on mainnet (or the reverse) is a blocking bug.
- Signing must go through `dw-vault`'s `VaultSigner` with a valid auth grant and respect lock state and auto-lock. Flag any new path that signs, exports or reveals keys without re-authentication where the existing flows require it.
- Secrets must not be logged, written to `app.sqlite`, sent to the renderer beyond explicit reveal flows, put in error messages, or kept in long-lived `String`s when a zeroizing type is available.
- Files under the wallet data directory go through `dw-fs`. Flag plain `std::fs` writes, permissive modes, or symlink-following opens on wallet data.
- Restore and import (`dw-compat`, recovery phrases, dumpwallet, wallet.dat, Platform address restore) must never strand funds: every balance the old wallet could spend must remain reachable.
- CoinJoin, InstantSend, ChainLock and coin-control changes need care around UTXO locking (`dw-appdb` UTXO locks): no double-spend of locked or mixing coins, no spending unconfirmed coins the flow says it won't.
- Masternode/governance: owner-side only. The wallet must never sign with an operator key or generate operator secrets outside the keychain reveal flow. ProTx payloads must match Dash Core's serialization for the targeted version (v24 shared masternodes included).

## Platform / DashPay Rules

- Engine behavior that depends on `platform-wallet`, `dash-sdk`, `dpp` or `key-wallet` should be checked against the pinned rev in `rust/Cargo.toml`, not assumed from upstream docs. A dependency bump needs a look at wire-format, proof and protocol-version changes.
- Platform features must be gated on the network's protocol version and show a clear "not active" state when the feature is off.
- DashPay contact requests, encrypted payloads and profile data: check who can decrypt what, and that contact xpubs/keys are derived per the DashPay DIPs.
- Identity credit top-up/withdraw and shielded transfers need balance checks, max-cost guards, and readable failure states.

## Tauri / Renderer Rules

- Business rules belong in the Rust engine. Flag TypeScript that re-implements validation, fee logic or key derivation instead of calling a `dw-app` command.
- New Tauri commands are a trust boundary: validate inputs in Rust, return typed errors, and do not widen capabilities or the allowlist without a reason.
- Deep links (`dash:`, `pay:`, `dashpay:`, `dashwallet:`) are untrusted input; they must open a confirmation, never act silently.
- UI-changing PRs must carry light and dark screenshots, with before/after from the exact base and head revisions, on the PR itself (`AGENTS.md`, "UI artifacts on every UI pull request"). Missing artifacts on a UI PR are a valid finding.
- Accessibility: keep Playwright/axe passing; flows need keyboard access, loading, error and retry states.

## Test Expectations

- Bug fixes in the engine should come with a regression test in the affected crate when practical.
- Funds-moving changes should be exercised in the regtest suites (`regtest/`, `dwcli` feature tests) or explain why not.
- Renderer changes should update vitest/Playwright coverage, including the fixture backend.

## Things Not To Flag

- The frozen Swift app not getting a feature the Tauri app gained; Swift is frozen by ADR 0003.
- Missing UI for operator tasks (ProUpServTx, ProUpRevTx, node administration); they are out of scope by design.
- Unsigned or ad-hoc-signed release binaries, lack of notarization, or the "no security review" disclaimer; these are documented known gaps.
- Git dependencies on the `PastaPastaPasta/platform` and `PastaPastaPasta/rust-dashcore` forks by themselves.
- RustSec ignores already listed with reasons in `rust/deny.toml`.
