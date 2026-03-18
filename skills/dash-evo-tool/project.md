# Dash Evo Tool — Project Understanding

## Overview

Dash Evo Tool (`dashpay/dash-evo-tool`, aka DET) is a cross-platform
GUI application (Rust + egui) for interacting with Dash Evolution
(Dash Platform). It enables DPNS username registration, contest
voting, state transition viewing, wallet management, identity
operations, and token management across Mainnet/Testnet/Devnet.

## Source Directory Structure

```
src/
├── app.rs              # AppState: owns screens, polls tasks, dispatches
├── main.rs             # Entry point, eframe setup
├── ui/                 # Screens and UI components
│   ├── components/     # Reusable UI widgets (AmountInput, MessageBanner, etc.)
│   ├── dpns/           # DPNS name registration + contest screens
│   ├── identities/     # Identity management screens
│   ├── wallets/        # Wallet screens (keys, balances, transfers)
│   ├── tokens/         # Token management screens
│   ├── document_query/ # Document query/inspection
│   └── ...             # Other screen modules
├── backend_task/       # Async business logic (one submodule per domain)
│   ├── identity/       # Identity creation, top-up, withdrawal, keys
│   ├── wallet/         # Wallet sync, send, receive, address derivation
│   ├── contested_names/# DPNS contest operations
│   ├── document/       # Document queries, state transitions
│   ├── tokens/         # Token creation, transfer, minting
│   └── error.rs        # TaskError — typed error envelope
├── model/              # Data types (amounts, fees, settings, models)
├── database/           # SQLite persistence (rusqlite), per-domain modules
├── context/            # AppContext: network config, SDK, DB, wallets, caches
│   ├── identity_db.rs  # Identity DB helpers
│   ├── wallet_lifecycle.rs  # Wallet load/unload
│   └── settings_db.rs  # App settings persistence
├── spv/                # Simplified Payment Verification (light wallet)
├── components/
│   └── core_zmq_listener  # Real-time Dash Core events via ZMQ
└── platform/           # Platform SDK wrapper layer
```

## Architecture Patterns

### App Task System (Critical)

The UI and async backend communicate through an action/channel pattern:

1. Screens return `AppAction` from `ui()` (e.g., `AppAction::BackendTask(task)`)
2. `AppState` spawns a tokio task → `AppContext::run_backend_task()`
3. Results return via tokio MPSC as `TaskResult` (Success/Error/Refresh)
4. Main `update()` loop polls receiver, routes to visible screen

Backend task enums: `BackendTask` has variants like `IdentityTask`,
`WalletTask`, `TokenTask(Box<TokenTask>)`, etc. Each has its own
`run_*_task()` method. Results are `BackendTaskSuccessResult` with
50+ typed variants.

### Screen Pattern

All screens implement `ScreenLike`:
- `ui(&mut self, ctx: &Context) -> AppAction`
- `display_task_result(&mut self, result: BackendTaskSuccessResult)`
- `refresh(&mut self)` / `refresh_on_arrival(&mut self)`
- `change_context(&mut self, app_context: &Arc<AppContext>)`

Root screens persist in `AppState.main_screens` (BTreeMap).
Modal/detail screens push onto `AppState.screen_stack`.

### UI Component Pattern

Lazy initialization with builder pattern:
- Private fields only
- Builder methods for config (`with_label()`, etc.)
- Response structs with `ComponentResponse` trait
- Self-contained validation
- Light + dark mode via `ComponentStyles`

### AppContext

`AppContext` (~50 fields), `Arc`-wrapped, shared everywhere:
- `sdk: RwLock<Sdk>` — Dash SDK (clone for async to avoid lock across await)
- `db: Arc<Database>` — SQLite persistence
- `wallets: RwLock<BTreeMap<...>>` — loaded wallets
- Cached system contracts (DPNS, DashPay, withdrawals, tokens)
- Per-network instances (mainnet always present, others on demand)

### Error Handling

`TaskError` (`src/backend_task/error.rs`) is a typed error envelope:
- `Display` → user-friendly text for `MessageBanner`
- `Debug` → technical details for logs
- Domain errors wired as `#[from]` variants
- Never store user-facing strings in error variant fields
- `BannerHandle::with_details(e)` for technical detail attachment

## Key Dependencies

- `dash-sdk` — Dash blockchain SDK (git dep from dashpay/platform)
- `egui/eframe 0.33` — Immediate mode GUI framework
- `tokio` — Async runtime (12 worker threads)
- `rusqlite` — SQLite with bundled library
- Rust edition 2024, minimum rust-version 1.92

## Build & Test

```bash
cargo build                    # Debug build
cargo test --all-features --workspace  # All tests
cargo clippy --all-features --all-targets -- -D warnings  # Lint
cargo +nightly fmt --all       # Format
```

Test types:
- Unit tests: inline `#[test]` in source files
- UI integration: `tests/kittest/` (egui_kittest)
- E2E: `tests/e2e/`

## Branch Model

- `master` — release-only, updated every few months
- `v1.0-dev` — active development branch (use as PR base)
- Conventional commit format required

## Platform Targets

Linux (x86_64/aarch64), Windows (x86_64), macOS (x86_64/aarch64
with code signing). Requires protoc v25.2+.
