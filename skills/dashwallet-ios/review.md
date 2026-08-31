# Dash Wallet iOS — Review Skill

## Review Posture

- Establish the PR's stated goal, base branch, and exact head SHA. Review the
  base-to-head diff and the relevant callers, not just isolated changed lines.
- Keep findings in scope: introduced by the PR, exposed by its new API/behavior, or
  required for the stated flow to work. Do not report unrelated legacy UIKit, migration
  debt, or broad modernization opportunities.
- Treat wallet value, mnemonic/keychain handling, authentication, network selection,
  transaction lifecycle, persistence migrations, and Swift/Rust boundaries as high-risk
  code.
- Verify behavior against current source and project configuration. Do not rely on stale
  line numbers or comments that describe intended behavior without an implementation.
- Build and runtime evidence must come from the exact reviewed head. A nearby branch,
  stale XCFramework, or old CI run is not evidence for the current code.

## Expected Validation

Use the workspace and the migration's working scheme. The SwiftDashSDK simulator
framework currently requires an arm64 destination:

```bash
pod install  # after Podfile/lockfile integration changes
xcodebuild -workspace DashWallet.xcworkspace -scheme dashpay -sdk iphonesimulator \
  -destination 'generic/platform=iOS Simulator' ARCHS=arm64 build
```

Do not substitute `DashWallet.xcodeproj` for the workspace. If SDK symbols or simulator
slices are missing, verify the sibling `platform` checkout and rebuild
`platform/packages/swift-sdk/DashSDKFFI.xcframework` rather than treating setup failure
as a code failure.

Run focused XCTest coverage when the unit-test target is operational. On branches where
its documented pre-existing breakage remains, require compile-ready tests for new logic
plus a clean `dashpay` build and a targeted simulator/device smoke of the affected flow.
A money-moving, migration, wipe, identity, deep-link, or SDK-lifecycle change needs
observed end-to-end behavior; static inspection alone is not enough.

Apply the configured formatter/linter to changed Swift or Objective-C files when
available. For project-file changes, confirm target membership and build settings for
every affected scheme/configuration. For localized resources, preserve the existing
encoding: main-app `.strings`/`.stringsdict` catalogs are UTF-8, while most WatchApp
interface catalogs are UTF-16LE.

## Wallet and Transaction Correctness

- Follow each spend from user input through amount parsing, network/address validation,
  authorization, input selection, fee computation, signing, confirmation UI, broadcast,
  and final state update.
- `prepare` and build/sign operations must not broadcast. Value transfer occurs only
  after explicit confirmation, and Cancel must leave the transaction unbroadcast.
- Confirmation UI must display the actual recipient, DASH amount, fee, total, and
  network used by the transaction that will be broadcast. Byte counts, estimates from a
  different path, and stale ViewModel values are not substitutes.
- Check DASH-versus-duff units and overflow/underflow at every boundary. Do not accept
  silent truncation, floating-point value transfer, negative-to-unsigned conversion, or
  fee arithmetic that can exceed the selected balance.
- Selected-input/sweep paths must preserve the caller's intended UTXO set and must not
  accidentally fall back to the standard sender.
- Reuse `WalletSendService`, `SwiftDashSDKTransactionSender`, `AuthenticationGate`, and
  `ScriptAddressCodec`; copied private variants frequently drift in timeout, biometric,
  validation, and fee behavior.
- Transaction and lock state should come from the SDK/runtime source of truth. Unknown
  metadata must not be presented as a plausible `.ok`, `.classic`, unlocked, or empty
  value.

## Seed, Wallet, Migration, and Wipe Safety

- Review `DASHSYNC_KEY_MIGRATION.md` as the contract for any legacy-keychain, wallet
  creation/import, active-wallet, or wipe change.
- Normal migration reads legacy DashSync mnemonic entries without deleting them, records
  per-wallet progress, and remains resumable after partial failures.
- New seed writes go through the host and SwiftDashSDK `WalletStorage`. Flag ad-hoc
  `WalletStorage()` reads, selection of `.first`, or wallet-ID assumptions that can
  choose the wrong seed in a multi-wallet install.
- Mainnet and testnet active-wallet mappings are independent. Creation, import,
  switching, recovery, mirroring, removal, and phrase display must resolve the active
  wallet for the current network.
- A live wallet must not be created before its mnemonic is durably stored and verified.
  Cleanup after a failed create must remove only state provisionally created by that
  attempt.
- Wipe and Remove flows fail closed: authorize the requested scope, preserve unrelated
  wallets and keychain services, wait for deletion completion, and report success only
  after the required SDK/app state is gone.
- Recovery authorization must reject partial keychain visibility and must not let one
  phrase authorize deleting a distinct stored wallet.
- Do not restore DashSync runtime imports, `DS*` wallet/transaction/identity objects,
  `DWEnvironment`, or a staged DashSync/SDK fallback. Only the documented compatibility
  migrator/wiper may understand the frozen legacy keychain layout.

## SwiftDashSDK / FFI Boundary

- Trace both sides of every changed Swift/Objective-C/Rust-facing API. Confirm generated
  bindings and the `DashSDKFFI.xcframework` version actually expose the used symbol and
  type shape.
- Validate pointer/buffer lifetime, ownership transfer, nullability, string/data
  encoding, integer width/signedness, enum evolution, and error conversion. A Swift
  `Data.withUnsafeBytes` pointer must not escape its closure unless the callee copies
  it.
- FFI errors, panics, nulls, and unavailable values must not be flattened into success
  or a plausible default. Preserve enough typed context for the UI and retry logic to
  react correctly.
- Respect SwiftDashSDK actor/thread requirements. Avoid blocking the main actor on
  semaphores, synchronous dispatch, long FFI calls, network operations, or polling
  loops.
- Check callbacks and continuations for exactly-once completion across success, error,
  cancellation, timeout, and deallocation paths. Verify callback queue assumptions
  before touching UIKit or `@MainActor` state.
- Lifecycle ownership belongs to the host/runtime/coordinator boundaries. Flag duplicate
  starts, use after shutdown, stale observers/tasks, or parallel owners that can race
  wallet/SPV teardown.
- SPV completion is `SyncingActivityMonitor.syncDone`; `waitForEvents` is the normal
  fully caught-up state. Code that waits for `.synced` can stall indefinitely.

## Swift Concurrency and UI

- New UI must be SwiftUI with a ViewModel. A SwiftUI `View` must not perform SDK/FFI
  calls, authentication, persistence, protocol fee math, chain mapping, or broadcasts.
- `ObservableObject` ViewModels that publish UI state should be `@MainActor`. Check
  detached tasks, callbacks, Combine delivery, and notification handlers before they
  mutate state.
- Ensure task lifetime follows the owning screen/service. Repeated `.task`/`onAppear`
  events must not create duplicate requests, transactions, observers, timers, or sync
  loops.
- Check closure captures and observation cleanup for retain cycles, lost completions,
  and callbacks into deallocated UI.
- Force unwraps are findings only when the PR can violate the precondition. Interface
  Builder outlets and lifecycle-initialized UI IUOs are accepted legacy patterns when
  initialization is guaranteed.
- Conditional compilation must parse and behave correctly in both enabled and disabled
  configurations. Pay special attention to `#if` inside ViewBuilder expressions,
  collection literals, boolean expressions, and exhaustive switches.
- Existing UIKit can be maintained. Do not demand a SwiftUI rewrite unless the PR adds a
  new screen or substantially reworks the UI in violation of the repository's explicit
  direction.

## Persistence, Notifications, and Data Modeling

- SQLite schema changes require a new timestamped migration and regression coverage for
  both fresh and upgraded stores. Never edit an already-shipped migration in place.
- Check transactionality and retry behavior when a flow spans keychain,
  SQLite/SwiftData, UserDefaults, and live SDK state. Partial failure must not create a
  seedless wallet, orphan active ID, false success, or irrecoverable metadata state.
- New SDK state needs a typed app-owned publisher/notification. Re-emitting legacy
  DashSync notification strings creates ambiguous ownership and can wake unrelated
  consumers.
- Comments must match executable code. If a described method, mirror write, validation,
  or confirmation does not exist and run, treat the behavior as absent.
- Reject stub-and-assert behavior: no discarded user input, no no-op action reported as
  complete, and no fabricated status/network/type used to make UI appear finished.
- New shared singletons in Infrastructure need clear lifecycle ownership and a protocol
  seam or documented reason; check whether an existing host/service should own the
  behavior instead.

## Network, Platform, and External Services

- Network selection must be explicit. Fresh installs default to mainnet; testnet is
  user-selected. Never use an unknown-network fallback that could sign, display, query,
  or broadcast on the wrong chain.
- Keep L1 Dash and Platform credits, addresses, keys, and fee models distinct. Identity
  funding and Platform transfer changes must preserve the chosen funding source through
  confirmation and execution.
- Treat DAPI, explorer, swap, Coinbase, Uphold, Maya, merchant, QR/deep-link, and
  payment-protocol inputs as untrusted. Validate response/request shape, URL
  scheme/host, address/network, numeric bounds, and authentication state before side
  effects.
- Verify external-service callbacks are idempotent and cannot double-submit after retry,
  navigation, or app foregrounding.
- Deep links and payment requests must not bypass the same validation, authorization,
  and confirmation gates used by in-app entry points.

## Project and Resource Integrity

- New source/resources/tests must have correct Xcode target membership. File presence in
  the repository does not guarantee compilation into each intended target.
- Keep `Podfile` and `Podfile.lock` aligned. `pod update` churn unrelated dependencies
  and should not accompany a focused change without justification.
- Preserve platform-specific post-install settings: iOS pods use
  `IPHONEOS_DEPLOYMENT_TARGET`; watchOS pods use `WATCHOS_DEPLOYMENT_TARGET`.
- All app/watch/extension targets share `MARKETING_VERSION`. Flag hardcoded plist
  versions or partial release bumps.
- Changes under a feature flag need validation with the relevant flag both present and
  absent, especially `DASHPAY` and Debug/TestNet behavior.
- User-visible strings belong in localization catalogs. Avoid converting WatchApp
  UTF-16LE catalogs while editing UTF-8 main-app localizations.
- Generated or ignored SwiftDashSDK FFI artifacts should be reproducibly rebuilt, not
  committed merely to make one local build pass.

## Test Expectations

- Pure model/formatting/validation changes: focused XCTest cases for normal, boundary,
  invalid, and failure inputs.
- Wallet send or fee changes: tests for units, insufficient funds, exact-balance/fee
  edges, selected inputs, Cancel, failed broadcast, and network mismatch, plus a focused
  runtime smoke.
- Migration/wipe changes: fresh install, one and multiple wallets, partial retry,
  unreadable keychain item, separate networks, authorization denial, and
  failure-before-success ordering.
- FFI/lifecycle changes: missing/error results, cancellation, repeated start/stop,
  callback queue, deallocation, and simulator/device architecture coverage as
  applicable.
- SwiftUI/ViewModel changes: state transition tests and observed UI behavior for
  loading, success, failure, cancellation, repeated appearance, and disabled controls.
- Persistence migrations: both empty-store creation and upgrade from the immediately
  preceding shipped schema.
- Bug fixes should add the narrowest regression test or executable check that would have
  caught the defect.

## Things Not To Flag

- Legacy UIKit, storyboards, Objective-C, or CocoaPods usage outside the PR's change
  solely because newer alternatives exist.
- Lifecycle-guaranteed outlets/IUOs, established Objective-C nullability patterns, or
  force unwraps whose precondition is proven by the changed flow.
- The arm64-only simulator build override while the shipped SwiftDashSDK XCFramework has
  that documented slice constraint.
- The ignored `DashSDKFFI.xcframework` being absent from git when supported build/setup
  paths regenerate it.
- The presence of frozen DashSync key names inside the dedicated migration/wipe
  compatibility code or documentation.
- Missing broad test infrastructure by itself when the branch has documented
  pre-existing test-target breakage. Require proportional compile-ready tests and direct
  behavior evidence for the PR instead.
- Formatting, naming, localization ordering, or generated project-file churn unless it
  changes behavior, target membership, resource encoding, build correctness, or
  security.
