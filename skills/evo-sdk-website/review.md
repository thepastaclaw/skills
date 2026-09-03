# Evo SDK Website — Review Skill

## Review Posture

- Review only the PR under consideration. Do not propose unrelated repo work unless explicitly requested.
- Determine the PR's intent first, then keep findings in scope: introduced by the PR, exposed by the PR's new behavior, or required for its stated goal.
- This is a **public** repo. Diffs, comments, and findings are world-visible; avoid revealing anything from other private repos in review text.
- Most PRs here are dependency bumps of `@dashevo/evo-sdk`, doc-generator changes, or test/UI tweaks. Favor concrete correctness, wire-protocol, deployment, and test-stability findings over style preferences.

## Expected Local Gates

Package manager is **Yarn 4 (Berry)** via Corepack; do not substitute `npm`.

```bash
corepack enable
yarn install
yarn generate          # regenerates public/docs.html and public/AI_REFERENCE.md from api-definitions.json
yarn check             # validates generated docs are current
yarn test:smoke        # Playwright smoke tests (CI runs these on every PR)
yarn test:queries      # Playwright query suite (CI runs these on every PR)
```

Optional (not run in CI, slow):

```bash
yarn test:transitions  # sequential, slow, hits real network with WIFs from test-data
yarn test              # everything
```

CI (`.github/workflows/test-sdk-site.yml`) runs on Ubuntu with Node 20 + Python 3.11. Transition tests are intentionally skipped in CI per platform issue #2736 — do not treat their absence from CI logs as a regression.

## SDK Bump Rules

- SDK version bumps (`@dashevo/evo-sdk`) are the most common change. For each bump, the reviewer should care about:
  - Does any `protocol_version` / wire-format assumption in `buildClientOptions()` (in `public/app.js`) still match what the live network advertises? If the SDK now defaults to a newer `PlatformVersion::latest()` than the network's active version, queries can fail with opaque decode errors. A pinned default is acceptable; an unintentional unpin is a real risk.
  - Have response shapes the SDK returns changed (e.g. camelCase vs `V0`-wrapped, `firstBlockTime` vs `first_block_time`)? Tests under `tests/e2e/queries/` assert on shape; out-of-date assertions are the bug, not the SDK.
  - Did the SDK rename or remove an export used by `public/app.js`'s top-level import? Vanilla `<script type="module">` will hard-fail on a missing export.
  - Was the `public/dist/evo-sdk.module.js` bundle expected to be refreshed? Bundle is normally not committed; if a PR commits one, confirm it matches the new SDK version.
- A pin on `protocol_version` should have a Follow-up note in the PR description for when the network upgrades. Flag if the pin is left without an obvious owner/follow-up.

## Vanilla JS / Frontend Rules

- No framework. Treat `public/app.js` as the single-page app's whole runtime — global functions, DOM access by id, switch statements. Do not request a React/Vue/etc. rewrite.
- Module imports in the browser go through `public/dist/evo-sdk.module.js`. Adding new imports requires the matching export to exist in that bundle (which comes from `dashpay/platform`'s `js-evo-sdk`).
- Validate that user-supplied input from the UI (identity IDs, contract IDs, document JSON, WIF strings) is normalized before being passed to SDK calls. Bad input should surface a readable error, not an unhandled rejection in the console.
- WIF private keys are entered in the browser by the user. Do not log them, persist them to `localStorage`/`sessionStorage`, send them anywhere except the SDK call, or echo them into generated HTML. Any new path that touches a WIF should be audited for accidental persistence/exposure.
- BigInt usage matters for credit/token amounts (`BigInt(amount)` in many calls); silent coercion to `Number` will lose precision on real-world values.

## `api-definitions.json` / Docs Rules

- `public/docs.html` and `public/AI_REFERENCE.md` are generated. If a PR hand-edits them without also updating `api-definitions.json` and/or running `yarn generate`, the next generator run will overwrite the change. Flag this and ask for the JSON to be the source of truth.
- New operations must add both the `api-definitions.json` entry and a handler in `callEvo()` in `public/app.js`. Missing either side is a real bug.
- Doc drift is detected by `yarn check`. If a PR changes `api-definitions.json` but not the generated artifacts, `yarn check` will fail — flag it.

## Playwright / Test Rules

- Use `tests/e2e/utils/sdk-page.js` (page object model) and `tests/e2e/fixtures/test-data.js` for new tests. Avoid inlining identity/contract IDs or selectors.
- Tests hit the live testnet by default. Do not add tests that depend on mainnet state without an explicit reason.
- New query-shape assertions should match what the SDK actually returns at the bumped version; copy real responses rather than guessing field names.
- Transition tests are sequential, slow, and skipped in CI; they are not a substitute for query-suite coverage of new behavior reachable from the UI.
- `playwright.config.ts` sets a 120s global / 30s action / 60s navigation timeout. Adding tight per-test timeouts that fight these defaults usually causes flakiness; raise concerns instead of papering over with longer waits.

## Python Generator / Scripts Rules

- `scripts/generate_docs.py` and `scripts/check_documentation.py` are Python 3.11. Keep them stdlib-only or document new dependencies; there is no `requirements.txt` workflow today.
- The generator must remain idempotent — running `yarn generate` twice should be a no-op diff.
- The `postinstall` hook runs `yarn generate`, so any change that makes generation depend on the SDK bundle being already present must handle the install-time ordering.

## Secrets / Public-Repo Rules

- This repo is public. Treat any committed secret (real WIF, mnemonic, API key, signed token) as a real leak and a blocking finding. `.env.example` placeholder values are fine.
- Test data in `tests/e2e/fixtures/test-data.js` may reference testnet identities/keys intentionally exposed for testing. Do not flag those unless a PR newly commits a *mainnet* key or a non-test secret.

## Things Not To Flag

- Absence of a frontend framework, bundler, or TypeScript across `public/`. The project is intentionally vanilla JS.
- The `postinstall` running `yarn generate`. It is intentional, not a supply-chain concern in this context.
- Skipped transition tests in CI. Intentional per platform issue #2736.
- Use of `python3 -m http.server` as the dev server. Intentional, not a production-server concern; the deployed site is static GitHub Pages.
- Pure regenerated/static doc churn in `public/docs.html` / `public/AI_REFERENCE.md` unless it indicates `api-definitions.json` and the generator have diverged.
