# Dash Evo Tool — Review Skill

## Code Style

- Rust edition 2024, MSRV 1.92
- `cargo +nightly fmt --all` enforced
- `cargo clippy --all-features --all-targets -- -D warnings` (warnings are errors)
- Conventional commit format: `area: short description`
  (e.g., `feat(tokens): add minting screen`, `fix(wallet): correct balance calc`)

## Test Expectations

- **Unit tests**: Inline `#[test]` for new logic, data structures
- **UI integration tests**: `tests/kittest/` for egui screen behavior
- **E2E tests**: `tests/e2e/` for full workflow scenarios
- New backend tasks should have corresponding test coverage
- Changes to consensus-adjacent logic (fee calc, state transitions)
  MUST have tests

## Error Handling Conventions

- All backend errors use `TaskError` variants — never raw strings
- `Display` is user-facing (shown in MessageBanner); `Debug` is for logs
- New error types → new `TaskError` variant with `#[source]` field
- For SDK errors: `Box<SdkError>` as source field type
- `BannerHandle::with_details(e)` for technical details — never in the user message
- Never store user-facing strings in error variant `String` fields

## User-Facing Message Rules

- Write for everyday users — no jargon (no "consensus error", "nonce",
  "state transition", "SDK", "RPC")
- Structure: *what happened* + *what to do*
- Every message must include a concrete self-service action
- No "contact support" — users must be able to self-resolve
- Calm, direct, brief tone — not apologetic or alarming
- Base58 IDs (contract/identity/document IDs) are allowed — they're
  user-meaningful handles, not jargon

## Architecture Rules to Enforce

- `&AppContext` as first parameter after `self` when methods take it
- Screen constructors handle errors internally via `MessageBanner` —
  keep `create_screen()` clean
- Lazy initialization for UI components (not eager)
- Private fields only on components — builder pattern for config
- `ComponentResponse` trait on response structs
- Clone SDK for async use — never hold `RwLock` across await points
- `MessageBanner::set_global()` for banners — no manual guard needed
- `take_and_clear()` to dismiss progress banners (not just `= None`)
- i18n-ready strings: no concatenated fragments, no positional assumptions,
  single extractable units with named placeholders

## Backport / Upstream Considerations

- Not applicable — DET is a standalone Dash project, not a fork

## Known Patterns

<!-- Populated by feedback loop — do not prefill -->

## Things NOT to Flag

<!-- Populated by feedback loop — do not prefill -->
