# rust-dashcore — Project Understanding

`dashpay/rust-dashcore` is the Rust implementation of Dash primitives and wallet/SPV libraries. It includes core transaction/block types, key-wallet/account management, SPV sync, RPC/client pieces, and FFI bindings used by mobile and other cross-language consumers.

## Important Areas

- `dash/` / core crates: Dash consensus primitives, transactions, blocks, hashes, serialization.
- `key-wallet/`: wallet/account/address-pool state, derivation metadata, transaction matching, UTXO and balance tracking.
- `key-wallet-manager/`: higher-level wallet manager, wallet events, mempool/block processing, event emission and persistence-facing APIs.
- `dash-spv/`: SPV sync pipeline, filters, block processing, network coordination.
- `dash-spv-ffi/`: C-compatible FFI surface for Swift/mobile and other non-Rust callers. ABI and callback payload compatibility matter here.

## Review Priorities

- Preserve wallet/account/address metadata across API seams. Raw addresses are often not enough when consumers persist hierarchical wallet state.
- Treat public Rust APIs and FFI callback payloads as compatibility surfaces. Identify source/ABI breaks explicitly and distinguish intentional from accidental breaks.
- Check no-std/std feature boundaries where relevant.
- Check serialization and persisted wallet state changes for migration/backward-compatibility risk.
- For address derivation/gap-limit changes, verify account type, pool type, index, derivation path, public key, and dedup semantics.
- For SPV/block processing changes, trace data from block/mempool detection through wallet state updates and emitted events.
