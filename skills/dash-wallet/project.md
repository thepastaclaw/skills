# Dash Wallet — Project Understanding

Dash Wallet is a multi-module Android application for Dash payments. The
`wallet` module is the production app; `common` contains shared components;
`features/` contains feature modules; `integrations/` contains third-party
exchange integrations; and `integration-android` is the embeddable payment
library. Kotlin is the primary language with legacy Java, Android Views and
Jetpack Compose, Hilt dependency injection, Room/SQLite persistence, Retrofit
and OkHttp networking, and dashj/bitcoinj-derived wallet and chain code.

The app uses MVVM. ViewModels expose one immutable `UIState` through
`StateFlow`, use private mutable state, and are normally `@HiltViewModel`.
Mainnet (`prod`) and `_testNet3` flavors have different signing, wallet-file,
and network behavior; never assume a testnet-only setting is safe in prod.

Build and test commands are documented in `CLAUDE.md`. The useful quick checks
are `./gradlew :wallet:compile_testNet3DebugKotlin` and
`./gradlew :wallet:test_testNet3DebugUnitTest`; full builds may require local
Firebase and integration properties. Inspect module Gradle files before
changing dependencies, variants, manifest components, or ProGuard rules.
