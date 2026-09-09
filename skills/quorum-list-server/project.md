# Quorum List Server — Project Understanding

## Overview

`dashpay/quorum-list-server` is a Rust Axum HTTP service that reads Dash Core
RPC data, caches it in process, and serves REST endpoints for Platform-relevant
LLMQ quorum state and Evo masternode metadata. Consumers such as Dashmate use it
as a bootstrap/cache source rather than calling Core RPC directly.

Default branch: `master`. Toolchain pin: Rust `1.91.0` (`rust-toolchain.toml`).
Edition: 2021.

## Source Layout

```
src/
├── main.rs              # Config load, cache warm + background refresh, Axum serve, Ctrl-C shutdown
├── api.rs               # Routes, response envelopes, host-override on /masternodes
├── config.rs            # TOML/env config, Network → LLMQ type/id + DAPI port, docker overrides
├── quorum_list.rs       # QuorumList / QuorumListEntry / QuorumMember models
├── quorum_loader.rs     # Core RPC: quorum listextended / quorum info / getblockcount
├── quorum_cache.rs      # Current + previous-height quorum caches; TTL background refresh
├── masternode.rs        # Core masternode list models → Evo-only response projection
├── masternode_loader.rs # Core RPC: masternode list
├── masternode_cache.rs  # Evo masternode cache + concurrent DAPI version probes
└── grpc_client.rs       # tonic GetStatus against Platform DAPI (TLS except regtest)
proto/platform.proto     # DAPI GetStatus surface compiled by build.rs / tonic-build
config.toml              # Local/runtime defaults (file-first load path)
Dockerfile               # Multi-stage release image
terraform/               # AWS testnet deploy modules (compute, LB, DNS, network)
```

## Runtime Architecture

1. `Config::load_from_env_or_file("config.toml")` — file wins when it parses;
   otherwise env vars / defaults.
2. `QuorumCache` and `MasternodeCache` are created, refreshed once at startup,
   then refreshed on background intervals.
3. HTTP handlers read only from cache. They do not call Dash Core or probe
   masternodes on the request path.
4. Axum serves with permissive CORS and response compression.

### HTTP surface

| Method | Path | Source |
|--------|------|--------|
| GET | `/health` | static OK |
| GET | `/quorums` | current quorum cache |
| GET | `/quorums/stats` | current quorum cache |
| GET | `/quorums/:hash` | current quorum cache by hex hash |
| GET | `/previous` | previous-height quorum cache |
| GET | `/masternodes` | Evo masternode cache (+ optional host override) |

Responses use `ApiResponse<T>`: `{ success, data, message }`.

### Network → LLMQ / DAPI mapping

`Network` selects the Platform LLMQ type and default DAPI HTTPS port:

- mainnet → `llmq_100_67` (type 4), port 443
- testnet → `llmq_25_67` (type 6), port 1443
- devnet → `llmq_devnet_platform` (type 107), port 1443
- regtest → `llmq_test_platform` (type 106), port 2443

### Quorum path

- RPC: `quorum listextended` (+ optional height) and `quorum info` for the
  network's LLMQ type/id; public keys are expected as 48-byte BLS keys.
- Cache: `current_quorums` and `previous_quorums` behind `RwLock`, refreshed on
  `quorum.cache_ttl_seconds` (default 60s) with a 30s refresh timeout.
- Previous height = current `getblockcount` − `previous_blocks_offset`.

### Masternode path

- RPC: `masternode list`, deserialized then filtered to `type == "Evo"`.
- Cache refresh (default every 10 minutes) concurrently probes each node's
  Platform HTTP port (or network DAPI default) via gRPC `GetStatus` to set
  `versionCheck` / `dapiVersion` / `driveVersion`.
- `POSE_BANNED` nodes are marked fail and not probed.
- Request reads never probe; `/masternodes` may rewrite the address host via
  `docker.address_host_override` / `ADDRESS_HOST_OVERRIDE`.
- `docker.version_check_host` / `VERSION_CHECK_HOST` can redirect probe targets
  (e.g. Docker Desktop `host.docker.internal`).

## Key Dependencies

- `axum` + `tower-http` (CORS, compression)
- `tokio`, `futures`
- `dashcore` / `dashcore-rpc` from `dashpay/rust-dashcore` tag `v0.40.0`
- `tonic` / `prost` + local `proto/platform.proto`
- `serde` / `serde_json` / `toml` / `hex` / `semver` / `chrono`

## Configuration knobs

Server: `server.host` / `server.port` (`API_HOST`, `API_PORT`).

RPC: `rpc.url` / `rpc.username` / `rpc.password` (`DASH_RPC_URL`,
`DASH_RPC_USER`, `DASH_RPC_PASSWORD`).

Quorum: `previous_blocks_offset`, `cache_ttl_seconds`
(`QUORUM_PREVIOUS_BLOCKS_OFFSET`, `QUORUM_CACHE_TTL_SECONDS`).

Network: top-level `network` or `DASH_NETWORK`.

Docker helpers: `version_check_host`, `address_host_override`.

## Build & Test

```bash
cargo build --locked
cargo test --locked
cargo clippy --locked --all-targets
cargo fmt
```

Unit tests currently live mainly in `config.rs`. PR validation commonly also
runs build/test/clippy locked. Deploy artifacts: Docker image + `terraform/`
for testnet infrastructure.
