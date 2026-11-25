# Dash Wallet Desktop — Project Skill

## Project Purpose

`PastaPastaPasta/dashwallet-desktop` is a public, experimental desktop Dash wallet for macOS, Windows and Linux: the DashPay mobile wallets (iOS and Android) adapted to the desktop. It covers Core send and receive, SPV sync, the shielded pool, Platform addresses, DashPay usernames, contacts and invitations, CoinJoin, and owner-side masternode and governance flows. Dash Core (dash-qt) stays the power-user tool; this wallet is not a replacement for running a node.

Repository: `PastaPastaPasta/dashwallet-desktop`, default branch `main`. Releases are tagged `v0.x.y`; every merge to `main` also publishes a preview build.

The repo is early: README says no security review has been done. It holds real users' recovery phrases, so wallet-safety bugs matter more than polish.

## Boundaries

- This repo is public. Do not expose private Dash/DCG context, private hostnames or IPs in GitHub review text.
- Review PRs only. Do not push code changes, branches, commits, or fix PRs unless explicitly requested.
- Treat recovery phrases, passphrases, private keys, WIFs, vault DEKs and wrap slots, auth grants, masternode keys, identity keys and DashPay contact encryption keys as sensitive.
- The authoritative product scope is `docs/design/DASHPAY.md` (DEC-01, §5a) and the task plan is `docs/design/ROADMAP.md`. Repo-wide agent rules are in `AGENTS.md` (mirrored in `CLAUDE.md`).

## Tech Stack

- UI: **Tauri 2** with a React + TypeScript renderer in `apps/desktop` (ADR `docs/adr/0003-ui-stack.md`, accepted 2026-10-11). pnpm, vite, vitest, Playwright + axe, tauri-driver E2E.
- Legacy UI: Swift (SwiftUI on macOS, a vendored SwiftCrossUI on Linux/Windows) over a UniFFI facade. **Swift is frozen** per ADR 0003: it is kept building and testing until Tauri parity and the `swift-final` tag, then deleted. New features belong in the Tauri app.
- Engine: a Rust workspace in `rust/` built on dashpay/platform's `platform-wallet`, `platform-wallet-storage` (SQLite), `dash-sdk`, `dpp`, `drive` (verify) and rust-dashcore's `dashcore` / `key-wallet`. These are pinned by git rev in `rust/Cargo.toml` to the `PastaPastaPasta/platform` and `PastaPastaPasta/rust-dashcore` forks; the sister checkouts of `dashpay/platform` and `dashpay/rust-dashcore` are at their default branches, so check the pinned rev when behavior depends on it.
- Toolchain pinned in `rust/rust-toolchain.toml`; `rust/deny.toml` holds the RustSec advisory ignores with reasons.

## Repository Layout

- `rust/crates/dw-engine` — tokio runtime and per-network sessions over platform-wallet + SqlitePersister; the core of the wallet.
- `rust/crates/dw-app` — the Tauri crate: commands, the event bridge, and generated TypeScript types (`apps/desktop/src/api/generated`, checked by CI with `DW_UPDATE_BINDINGS=1`).
- `rust/crates/dw-vault` — secret vault: DEK + wrap slots, lock states, auth grants, `VaultSigner`.
- `rust/crates/dw-fs` — owner-only directories and files, changed through descriptors that never follow a swapped-in symlink.
- `rust/crates/dw-appdb` — per-network `app.sqlite`: address book, labels, tx metadata, receive requests, UTXO locks, settings.
- `rust/crates/dw-coinjoin`, `dw-p2p` — CoinJoin client and its own outbound P2P sessions.
- `rust/crates/dw-governance`, `dw-protx` — governance over SPV; owner-side ProTx builders and v24 shared-masternode codecs.
- `rust/crates/dw-psbt`, `dw-message`, `dw-units`, `dw-uri`, `dw-compat` — PSBT, signmessage/verifymessage, amount formatting, `dash:` URIs and deep links, Dash Core wallet import (BIP39 quirks, dumpwallet, descriptor wallet.dat).
- `rust/crates/dw-console` — dash-qt-style RPC console over dw-engine.
- `rust/crates/dw-desktop` — OS services: single instance, autostart, URI scheme registration, notifications.
- `rust/crates/dw-ffi` — UniFFI facade for the frozen Swift package.
- `rust/bin/dwcli` — CLI used by tests and regtest suites.
- `apps/desktop` — the Tauri renderer (React/TS), e2e and e2e-tauri suites, fixture backend.
- `apps/macOS`, `Sources/`, `Package.swift`, `Vendor/swift-cross-ui` — the frozen Swift app and its vendored SwiftCrossUI patches (`Vendor/PATCHES.md`).
- `regtest/` — Docker-based dashd regtest harness and pytest suites run nightly.
- `ci/`, `.github/workflows/` — CI scripts; `docs/ci.md` documents every job.
- `docs/design/` — per-task design notes (DP*, MG*, U-*, E0-*), `DECISIONS-PENDING.md`, `UX-SPEC.md`.

## Architecture Notes

- Business logic belongs in the Rust engine, not the renderer. The renderer talks to `dw-app` commands only; a fix that duplicates engine rules in TypeScript is in the wrong layer.
- Secrets live in `dw-vault`. Signing goes through `VaultSigner` with an auth grant; nothing else should hold decrypted seed or key material longer than needed or hand it to the renderer.
- Wallet data files go through `dw-fs` (owner-only permissions, no symlink following). Writing wallet data with plain `std::fs` paths is a regression.
- Network choice (mainnet/testnet) is per session and applied on restart; state must not leak across networks (`app.sqlite` is per network).
- Platform features are gated on the network's Platform protocol version (for example the shielded pool). New Platform features need the same gate and a user-visible "not active" state.
- Product scope: owner-side masternode flows are in scope; operator tasks (ProUpServTx, ProUpRevTx, generating operator BLS secrets the wallet does not derive, running nodes) are deliberately out of scope. The wallet never signs with an operator key.
