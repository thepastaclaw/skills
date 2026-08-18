# Docs Platform — Project Skill

## Project Purpose

`dashpay/docs-platform` is the public Sphinx documentation site for Dash Platform. Reviews may run against contributor forks such as `thephez/docs-platform`; the fork does not change the public-repository trust model. The site publishes tutorials, explanations, protocol and API reference material, SDK documentation, and browser-based examples to Read the Docs.

## Boundaries

- This repository is **public**. Diffs, review comments, generated previews, and committed content are world-visible. There are no intentional committed credentials; treat a real key, mnemonic, WIF, token, or private endpoint as a leak.
- Review PRs only. Do not push fixes or modify the reviewed checkout unless explicitly requested.
- Documentation often describes behavior implemented elsewhere. Tutorial code is synchronized from `dashpay/platform-tutorials`; DAPI endpoint details and source links must be checked against the referenced Platform branch/protobuf source rather than inferred.
- Keep review scope proportional to the PR's stated intent. Pre-existing documentation debt is context, not a finding, unless the PR relies on it or makes it materially worse.

## Tech Stack

- Documentation: Markdown with MyST Parser, plus Sphinx directives and occasional reStructuredText.
- Site build: Sphinx 8.1.3 with `pydata_sphinx_theme`, configured in `conf.py` and pinned in `requirements.txt`.
- Hosting: Read the Docs on Ubuntu 22.04 with Python 3.10 and Node.js 22, configured by `.readthedocs.yml`.
- Browser behavior: framework-free JavaScript and CSS under `_static/`.
- Interactive tutorial SDK: `@dashevo/evo-sdk`, bundled for browsers by esbuild from `scripts/evo-sdk-entry.js`; npm dependencies are pinned by `package-lock.json`.

## Repository Layout

- `index.md`, `docs/index.md` — root and Platform documentation toctrees.
- `docs/tutorials/` — step-by-step guides; embedded code may be synchronized from `platform-tutorials`.
- `docs/explanations/`, `docs/reference/`, `docs/protocol-ref/`, `docs/sdk-rs/`, `docs/resources/` — conceptual, API/protocol, SDK, and supporting content.
- `conf.py` — Sphinx extensions, exclusions, theme, sidebars, static assets, and browser script registration.
- `_templates/` — custom Jinja templates, including sidebar behavior.
- `_static/` — CSS, JavaScript, images, standalone lite demos, and generated browser assets.
- `_static/js/interactive-tutorial.js` — interactive runner and its explicit operation allow-list.
- `_static/vendor/evo-sdk.js` — generated, gitignored SDK bundle; it must be produced during every relevant local and hosted build.
- `scripts/tutorial-sync/` — tutorial code synchronization/checking against `dashpay/platform-tutorials`.
- `scripts/sync_sidebar.py` — updates the custom sidebar after navigation changes.
- `.readthedocs.yml`, `Makefile`, `make.bat` — hosted and local build entry points.

## Architectural Notes

- `make html` depends on the `sdk` target. A fresh local checkout first needs `make sdk-install`; Read the Docs performs the equivalent `npm ci` and `npm run build:sdk` in `pre_build`.
- The generated Evo SDK bundle is intentionally not committed. Build correctness therefore depends on all supported build paths producing it before Sphinx copies static assets.
- The interactive runner is loaded as a classic deferred script, then lazily imports the local ESM SDK bundle relative to its own script URL.
- Interactive blocks in tutorial Markdown use MyST `{raw} html` with declarative `data-operation`, `data-param`, and `data-renderer` values. The current runner maps operation names to fixed JavaScript functions; displayed browser code is derived from the same function object but is not executed from the DOM.
- The operation map is deliberately read-only and uses `EvoSDK.testnetTrusted()`. Adding arbitrary code evaluation, state transitions, mainnet behavior, or private-key input would change the feature's trust model and requires explicit design and security scrutiny.
- The standalone `*-lite.html` examples are separate browser applications and currently load their pinned Evo SDK from `esm.sh`; do not assume they share the interactive runner's local-bundle path.
- New pages must be included in the appropriate toctree. Navigation changes may also require `python scripts/sync_sidebar.py` so the custom sidebar remains synchronized.

## Important Trust Boundaries

- Public PR Markdown/raw HTML/static JavaScript → executable content in readers' browsers and Read the Docs previews.
- Tutorial form values → allow-listed Evo SDK calls → public Dash Platform testnet services.
- Remote SDK/testnet responses → DOM rendering. Keep response and error data on text-safe rendering paths; do not turn it into executable HTML.
- npm registry packages and `package-lock.json` → esbuild → generated browser bundle → published documentation.
- `platform-tutorials` source checkout → sync script → committed tutorial code blocks.
- `pull_request_target` workflows → GitHub token and PR metadata. Such workflows must not execute or source untrusted PR-head code with privileged credentials.
