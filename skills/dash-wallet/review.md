# Dash Wallet Review Guidance

Review Android changes with emphasis on user funds, transaction construction
and signing, wallet encryption and backup/recovery, address/network selection,
and persistence migrations. Trace code through ViewModel, repository/service,
dashj wallet/chain APIs, and UI rather than reviewing a single screen in
isolation. Mainnet and `_testNet3` flavors must keep their intended separation;
configuration or resource changes that can route prod to testnet, expose wallet
files, disable certificate checks, or skip signature/fee validation are
blocking.

For asynchronous work, check lifecycle cancellation, StateFlow state
consistency, threading, duplicate submissions, and error recovery. For Room or
serialized wallet data, verify migration/version compatibility and that old
wallets remain readable. For Retrofit/OkHttp and exchange integrations, check
TLS, authentication, timeouts, parsing of untrusted responses, and secrets in
logs or resources. For UI and deep links, check authorization, sensitive-data
exposure, accessibility regressions, and configuration changes across device
rotation and process death.

Keep findings tied to behavior introduced by the PR. Do not flag intentional
test fixtures or the documented testnet world-readable wallet behavior unless
the change leaks it into production.

## State and Lifecycle Checklist

Most wallet bugs that reach maintainers only show up when you list state
sequences. For each persisted state and each event the PR touches:

- List every state × transition the PR names, plus re-entry, retry, offline,
  locked device / Keystore unavailable, first launch, background launch,
  restore, wipe, and removal while an operation is in flight.
- Trace nil / unknown / unreadable into every aggregate and decision: unknown
  must not become zero, absent, or "no wallet"; a read failure is not absence.
- Check early-return order against teardown and guards (teardown before a
  guard, a guard released before the work it protects).
- Check snapshot-then-store paths for lost updates and stale overwrites (a
  newer state overwritten by a re-emit of an older one).
- For a new launch or background entry point, list the existing reads that
  now run in that context and check each.
- For a gate that blocks the user (a hold, a migration card), list every
  state in which it can never release.

