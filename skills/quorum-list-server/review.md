# Quorum List Server — Review Skill

## Review Posture

- Review the base-to-head diff against exact head SHA. Trace RPC → cache → HTTP
  (and gRPC probe) paths for changed behavior; do not review only the hunk.
- Keep findings in scope of the PR. Pre-existing logging noise, broad refactors,
  or infra polish belong in `out_of_scope_findings` unless the PR breaks them.
- Treat this as a cache/API trust boundary in front of Dash Core and Platform
  nodes: wrong data, stale success, host override mistakes, and probe/DoS
  behavior matter more than style.

## Validation Expectations

```bash
cargo build --locked
cargo test --locked
cargo clippy --locked --all-targets
cargo fmt --check
```

Require tests for changed serialization/compatibility, cache freshness
semantics, host-override behavior, and RPC/gRPC failure handling when those
surfaces move. Existing compiler/clippy warnings are not automatic findings
unless the PR introduces new ones or claims a clean tree.

## Correctness Focus

- **Cache vs request path:** HTTP handlers must remain pure cache reads.
  Reintroducing RPC or version probes onto `/quorums*` or `/masternodes`
  request handling is a regression.
- **Refresh failure semantics:** failed background refreshes should preserve
  the last good cache (and any published freshness timestamp) unless the PR
  deliberately documents a different contract. Do not serve empty success
  after a failed refresh.
- **Network mapping:** LLMQ type/id and DAPI port must stay consistent with
  `Network`. Silent wrong-network defaults are high impact.
- **Quorum parsing:** 32-byte quorum hashes and 48-byte public keys; selecting
  the wrong LLMQ bucket from `listextended` is a functional break for
  consumers.
- **Masternode projection:** only `Evo` nodes belong in `/masternodes`.
  Fields taken from Core's deterministic masternode list (including Platform
  identity/ports/addresses when exposed) must not be invented when absent.
- **Host overrides:** `address_host_override` and `version_check_host` must
  preserve ports, apply to every address field the PR claims they cover, and
  must not leak override behavior into unrelated networks accidentally.
- **API compatibility:** keep `{success,data,message}` envelope and existing
  field names unless the PR is an intentional break with consumer updates.
  Additive optional fields should use `skip_serializing_if` / omission rather
  than fabricated defaults.
- **Concurrency:** `RwLock` cache updates, `Arc` sharing, and background
  `tokio::spawn` refresh loops — watch for lost updates, holding sync locks
  across `.await`, and startup serving before first successful populate.

## Security / Trust Boundaries

- Dash Core RPC credentials and URL are sensitive config; do not log passwords
  or echo secrets in error/API responses.
- Outbound gRPC `GetStatus` talks to masternode-advertised hosts/ports (or
  configured overrides). Treat probe targets as potentially slow/hostile:
  timeouts, concurrency bounds, and TLS assumptions (HTTPS + native roots
  except regtest HTTP) matter.
- Permissive CORS and public read APIs are expected for this service; flag new
  write/admin surfaces, SSRF-like redirect of probes, unbounded allocation from
  RPC JSON, or response content that unexpectedly exposes RPC internals.
- Compression and large `/masternodes` payloads: watch for memory amplification
  or unbounded list growth from Core responses.

## Rust Quality Notes

- Prefer typed errors over growing `Box<dyn Error>` at module boundaries when
  the PR touches error paths materially.
- Avoid new `unwrap`/`expect` on RPC/JSON/cache lock paths in serving code.
- Keep blocking `dashcore_rpc` client work off the request path; if moved,
  it needs explicit `spawn_blocking` / timeout consideration.
- Match existing module split: loader (I/O) vs cache (state) vs api (HTTP).

## Things Not To Flag

- README lagging newly added endpoints when the PR does not claim docs
  completeness and code/tests define the contract (prefer noting only if the
  PR ships user-facing docs that contradict code).
- Verbose `println!` / emoji operational logging already present in cache/gRPC
  paths, unless the PR makes it materially worse or leaks secrets.
- Terraform/Docker/deploy choices outside the changed surface.
- Permissive CORS by itself — it is part of the current service design.
- Dependency pins on `dashpay/rust-dashcore` / tonic versions unless the PR
  changes them unsafely or breaks build reproducibility (`--locked`).
