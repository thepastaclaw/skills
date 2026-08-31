# Dash Wallet iOS — Project Understanding

## Overview

Dash Wallet iOS (`dashpay/dashwallet-ios`) is a non-custodial Dash wallet for iPhone,
with companion watch and Today extensions. It provides SPV wallet synchronization and
payments, DashPay identities and contacts, CoinJoin, wallet creation/recovery, merchant
discovery, swaps, and integrations with external services.

The application is migrating its live wallet and Platform behavior from the legacy
DashSync pod to the Rust-backed SwiftDashSDK. The app-side integration boundary is under
`DashWallet/Sources/Infrastructure/SwiftDashSDK/`; the SDK itself is supplied from
`dashpay/platform/packages/swift-sdk` as a local Swift package plus
`DashSDKFFI.xcframework`.

## Repository Layout

- `DashWallet/` — main application, resources, localizations, data models, and launch
  configuration.
- `DashWallet/Sources/Application/` — application-level lifecycle and synchronization
  monitoring.
- `DashWallet/Sources/Infrastructure/` — authentication, database, networking, currency,
  and SwiftDashSDK adapters.
- `DashWallet/Sources/Models/` — wallet-facing domain models and integrations such as
  CoinJoin, DashConnect, payment protocols, swaps, taxes, Uphold, and Coinbase.
- `DashWallet/Sources/UI/` — mixed legacy UIKit and modern SwiftUI feature code.
- `DashWalletTests/` — XCTest unit and integration coverage.
- `DashWalletScreenshotsUITests/` — screenshot UI tests.
- `TodayExtension/`, `WatchApp/`, `WatchApp Extension/` — companion targets.
- `DashWallet.xcworkspace` — canonical CocoaPods workspace; do not build the project
  file directly.
- `DashWallet.xcodeproj/project.pbxproj` — target membership, build settings, local
  Swift package references, and shared version settings.
- `Podfile`, `Podfile.lock` — CocoaPods dependencies for the app and extension targets.
- `DASHSYNC_KEY_MIGRATION.md` — frozen upgrade contract for importing legacy mnemonic
  entries.

## Languages and Frameworks

- Swift and Objective-C, with bridging headers in both directions.
- SwiftUI for new UI; UIKit and storyboards remain in the legacy application.
- Combine and Swift concurrency for state propagation and asynchronous work.
- SwiftData in the SwiftDashSDK integration and SQLite/SQLiteMigrationManager for
  app-owned data.
- XCTest for unit and UI tests.
- CocoaPods plus a local Swift package dependency on SwiftDashSDK.
- Rust-backed wallet, SPV, Platform, and cryptographic behavior accessed through
  `DashSDKFFI.xcframework`.

The iOS deployment target is 18.0. The watch targets retain their separate watchOS
deployment setting.

## Targets and Feature Flags

- `dashpay` is the working DashPay-enabled scheme during the SwiftDashSDK migration.
- `dashwallet` is the main app scheme but is not necessarily kept green on migration
  branches.
- `DASHPAY` enables identity, contacts, invitations, usernames, and governance features.
- `DASH_TESTNET` is not present in the `dashpay` Debug configuration.
  Development-only behavior shared by both schemes must use the repository's
  established `DEBUG || DASH_TESTNET` policy rather than assuming `DASH_TESTNET`
  covers Debug.
- Mainnet is the fresh-install default. Testnet requires an explicit user switch
  recorded by `WalletEnvironment`.

All targets use the shared `MARKETING_VERSION` build setting. Release changes must keep
app, extension, and watch target versions aligned instead of hardcoding plist values.

## SwiftDashSDK Architecture

`SwiftDashSDKHost` owns SDK startup and wallet creation/import.
`SwiftDashSDKWalletRuntime`, `SwiftDashSDKWalletState`, and `SwiftDashSDKSPVCoordinator`
own runtime lifecycle, published wallet state, and chain synchronization.
Feature-specific coordinators cover identity registration, profile updates, Platform
addresses, invitations, voting, and related DashPay behavior.

Important invariants:

- Do not restore DashSync runtime objects, imports, or fallback paths. Compatibility
  reads of DashSync-owned keychain records are confined to the frozen migrator/wiper
  contract.
- Resolve the active wallet by network through `SwiftDashSDKHost` and
  `WalletEnvironment`; never pick the first stored mnemonic or managed wallet when
  multiple wallets can exist.
- New mnemonic writes use SwiftDashSDK `WalletStorage`. The iOS keychain/device passcode
  is the security boundary; there is no app-side PIN encryption layer for the seed.
- A migration run is resumable and must not delete legacy recovery records during
  ordinary startup. Explicit Remove/Delete All flows may delete only the authorized
  legacy mnemonic accounts described by `DASHSYNC_KEY_MIGRATION.md`.
- Wallet wipes are fail-closed. Authorization, legacy cleanup, SDK mnemonic deletion,
  managed-wallet deletion, and app-state reset must preserve their documented ordering
  and must not report success after a partial failure.
- SPV completion is represented by `SyncingActivityMonitor.syncDone`. The underlying SPV
  steady state is `waitForEvents`; code must not wait for a transient `.synced` state.

## Money-Movement Boundaries

- Standard spends go through `WalletSendService`; selected-input and sweep flows go
  through `SwiftDashSDKTransactionSender`.
- Preparation/build-and-sign must not broadcast. Broadcast occurs only after explicit
  user confirmation.
- Authentication uses the shared `AuthenticationGate`; independent PIN/biometric
  continuations can hang or bypass established policy.
- Treat DASH/duff conversion, fee calculation, selected inputs, change, and the
  distinction between L1 Dash and Platform credits as correctness-critical.
- Network mapping must be explicit and fail closed. A missing or unknown chain must not
  silently fall back to testnet or mainnet.

## UI and Architecture Direction

All new UI is SwiftUI-first with an `@MainActor` `ObservableObject` ViewModel. Views
render state and route user intent; SDK/FFI calls, authentication, fee math, network
mapping, protocol constants, persistence, and money movement belong in ViewModels or
services. Existing UIKit can be maintained, and a thin `UIHostingController` may bridge
into legacy navigation, but substantial rework should move the screen toward SwiftUI
rather than add new storyboards, XIBs, or UIKit UI logic.

Prefer existing shared boundaries over copy-then-adapt. In particular, reuse
`AuthenticationGate`, `ScriptAddressCodec`, wallet/network resolution through the
host/environment, and established persistence services. New infrastructure singletons
need a clear ownership reason or protocol seam.

## Persistence and Upgrade Compatibility

- SQLite schema changes require a new timestamped migration under
  `DashWallet/Sources/Infrastructure/Database/Migrations.bundle/`; do not rewrite an
  already-shipped migration.
- Core/SwiftData and keychain operations can represent money or seed ownership and must
  preserve atomicity and retry behavior.
- Legacy notification names must not be re-emitted for new SDK state. Use a new typed
  publisher or app-owned notification.
- Unknown transaction or remote state should remain unknown. Do not fabricate plausible
  defaults such as a transaction type, success status, empty address list, or network.

## Build and Validation

Install dependencies with `pod install` after Podfile changes; avoid an unrelated
`pod update`. Always build the workspace. The canonical migration build is:

```bash
xcodebuild -workspace DashWallet.xcworkspace -scheme dashpay -sdk iphonesimulator \
  -destination 'generic/platform=iOS Simulator' ARCHS=arm64 build
```

The local SwiftDashSDK dependency expects sibling checkouts of `platform` and
`dapi-grpc`. Its ignored `DashSDKFFI.xcframework` must be rebuilt from
`platform/packages/swift-sdk` when SDK symbols or slices are stale.

The unit-test target may have pre-existing migration-branch breakage. New tests should
still be compile-ready; validation for affected runtime behavior requires a clean
`dashpay` build and a focused testnet/mainnet smoke of the changed flow. Once the target
is usable, `fastlane test` runs the XCTest suite.

Optional repository tools include SwiftFormat, SwiftLint, clang-format for Objective-C,
and BartyCrouch/localization checks.

## Security and Trust Boundaries

- Mnemonics, PIN/authentication state, wallet IDs, keychain records, signing keys,
  selected UTXOs, and wipe authorization are sensitive.
- QR codes, deep links, payment requests, pasted addresses, API responses, Platform/DAPI
  data, and external-service callbacks are untrusted input.
- Swift/Objective-C/Rust crossings require correct ownership, nullability, lifetime,
  queue, actor, encoding, and error semantics.
- External integrations must preserve token handling, redirect/deep-link validation, TLS
  expectations, and explicit user confirmation before value transfer.
- Chain/network selection, transaction signing, fee presentation, and broadcast are
  high-impact boundaries where a plausible fallback can cause irreversible loss.
