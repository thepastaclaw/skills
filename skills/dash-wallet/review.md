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
