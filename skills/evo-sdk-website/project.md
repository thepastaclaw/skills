# Evo SDK Website — Project Skill

## Project Purpose

`dashpay/evo-sdk-website` is an interactive documentation and testing site for the Dash Platform Evo JS SDK (`@dashevo/evo-sdk`, sourced from `dashpay/platform`'s `packages/js-evo-sdk`). It lets users execute queries and state transitions against the live testnet/mainnet directly from a browser UI without writing code, and doubles as the auto-generated reference docs surface for the SDK.

Live site: https://dashpay.github.io/evo-sdk-website/ (deployed via GitHub Pages from `master`).

## Boundaries

- This repo is **public**, unlike `dash-platform-website`. Assume diffs, comments, and findings are visible to the world. There are no intentional committed credentials here — treat any committed secret as a real leak.
- Review PRs only. Do not push code changes, branches, commits, or fix PRs against this repo unless explicitly requested.
- The SDK itself (`@dashevo/evo-sdk`) lives in `dashpay/platform`. This repo only consumes the published npm build via `package.json` and a copied `public/dist/evo-sdk.module.js` bundle. PRs here typically bump the SDK version and adjust callers/tests — they do not modify SDK internals.

## Tech Stack

- Pure vanilla JavaScript frontend in `public/` — no React, no framework, no bundler. Modules are loaded directly via `<script type="module">` from `public/dist/evo-sdk.module.js`.
- Package manager: **Yarn 4.12.0 (Berry)** via Corepack. `packageManager` field is pinned in `package.json`.
- Node `>=18` (CI uses Node 20).
- E2E tests: **Playwright** (`@playwright/test` 1.56.0). No unit-test framework.
- Doc generation: **Python 3.11** scripts (`scripts/generate_docs.py`, `scripts/check_documentation.py`).
- Local dev server: `python3 -m http.server 8081` serving `public/`.
- CI: GitHub Actions on Ubuntu (`.github/workflows/test-sdk-site.yml`). PRs run smoke + query Playwright projects against the local Python server; transition tests are skipped in CI (too slow due to platform issue #2736). Pages deploy happens on `master`.

## Repository Layout

- `public/index.html` — Main interactive SDK testing interface.
- `public/app.js` — Main app logic: SDK client management, query/transition dispatch via a `callEvo()` switch, dynamic UI (~3600 lines).
- `public/api-definitions.json` — Single source of truth for all SDK operations exposed by the site (~2700 lines). Doc HTML and `AI_REFERENCE.md` are generated from it.
- `public/docs.html`, `public/AI_REFERENCE.md` — Generated outputs; do not hand-edit.
- `public/dist/evo-sdk.module.js` — Bundled SDK, copied in from `platform/packages/js-evo-sdk/dist`. Not checked in to the working tree under normal flow.
- `public/service-worker-simple.js` — PWA-ish service worker.
- `scripts/generate_docs.py` — Generator: reads `api-definitions.json`, writes `docs.html` and `AI_REFERENCE.md`, copies SDK bundle.
- `scripts/check_documentation.py` — Validates docs are current.
- `tests/e2e/` — Playwright suites: `smoke/`, `queries/`, `transitions/`, plus `utils/` (page object `sdk-page.js`, `parameter-injector.js`, `base-test.js`) and `fixtures/test-data.js`.
- `playwright.config.ts` — Defines four projects: `site-tests`, `smoke-tests`, `parallel-e2e-tests`, `sequential-e2e-tests` (transitions, non-CI only). Global 120s timeout, 30s action, 60s navigation.
- `.github/workflows/` — `deploy-pages.yml`, `test-sdk-site.yml`, `update-evo-sdk.yml` (daily SDK bump automation).

## Architectural Notes

- The site is a thin demo/test harness. The interesting behavior is almost always: "did the SDK version bump break anything?" or "does the wire-protocol assumption still match the live network?"
- SDK protocol-version pinning matters: the SDK's `buildClientOptions()` can be told to fix a `protocol_version`, which controls which wire format the SDK emits (e.g. V0 vs V1 `GetDocumentsRequest`). The live network's active platform version dictates what is actually accepted. Mismatches surface as opaque decode failures like `"could not decode data contracts query"`.
- State transitions in the UI accept a user-supplied WIF private key entered in the browser. That key never leaves the browser — there is no backend — but the in-page handling still matters for UX and for avoiding accidental persistence/exposure.
- `api-definitions.json` is the contract between the UI and the generator. Adding a new operation requires touching both the JSON and the `callEvo()` switch in `app.js`, then regenerating docs with `yarn generate`.
- The `update-evo-sdk.yml` workflow opens automated PRs to bump the SDK; review them like any other dependency-bump PR, but pay extra attention to wire-protocol/version-pin assumptions.

## Sensitive Data Policy for Reviews

This repo is public and contains no intentionally committed credentials. Treat any committed secret (API key, WIF, mnemonic, `.env` not `.env.example`) as a real issue worth flagging. `.env.example` placeholders are fine.

## Important Trust Boundaries

- Browser UI ↔ user-supplied private key WIF input for signing state transitions.
- Browser SDK client ↔ live testnet/mainnet DAPI endpoints (real funds on mainnet).
- `api-definitions.json` ↔ generated `docs.html` / `AI_REFERENCE.md` (drift is a correctness bug, not a security one).
- Playwright test fixtures ↔ testnet identities and contracts referenced in `tests/e2e/fixtures/test-data.js`.
