# Dash Platform Website — Review Skill

## Review Posture

- Review only the PR under consideration. Do not propose or perform unrelated repo work unless explicitly requested.
- Determine the PR's intent first, then keep findings in scope: introduced by the PR, exposed by the PR's new behavior/API, or necessary for the PR's stated goal.
- This is a private repo with intentional committed testing secrets; do not flag committed `.website-config.json` credentials solely because they exist.
- Favor concrete correctness, data-loss, security, deployment, and cross-platform issues over style preferences.

## Expected Local Gates

Root gates:

```bash
npm ci
npm run typecheck
npm test
```

Useful focused gates:

```bash
npm run typecheck --workspace @dash-website/core
npm run typecheck --workspace @dash-website/scripts
npm run typecheck --workspace @dash-website/gateway
npm run typecheck --workspace @dash-website/deploy-tool
npm test --workspace packages/deploy-tool/client
```

CI runs root typecheck/test on Ubuntu, macOS, and Windows with Node 20. Be suspicious of POSIX-only shell/path/process assumptions unless they are intentionally launcher-specific.

## TypeScript / Node Rules

- Keep TypeScript strictness intact; avoid `any`, broad casts, and swallowed errors unless the boundary is explicitly validated.
- Validate and normalize external input from HTTP routes, filesystem scans, site configs, environment variables, and third-party APIs.
- Preserve useful error causes and avoid returning raw technical details or secrets to the browser.
- Do not introduce unbounded filesystem scans, recursive copies, upload loops, or in-memory buffers over user-controlled content without limits/backpressure.
- Use atomic filesystem helpers for writes that update manifests/config/state where partial writes would corrupt user workflow.
- Use path normalization and containment checks for any user-selected folder, upload source, extraction target, or remote path construction.

## Frontend / DWT Rules

- Client code should route API calls through the existing API wrapper and handle loading, cancellation, error, and SSE lifecycle states.
- Do not expose secrets by default. Reveal flows should be explicit and intentional.
- UI workflows that mutate or publish content need clear confirmation/progress/error states.
- Component tests are expected for new non-trivial UI behavior, especially setup, settings, build workflow, page selection, and error states.

## Gateway / CDN / SSH Rules

- Gateway and CDN code crosses remote operational boundaries. Check command construction, path handling, idempotency, retries, rollback/error paths, and secret leakage.
- SSH command execution must not concatenate untrusted values into shell commands without quoting/validation.
- Cloudflare/R2 code should handle pagination, non-2xx responses, eventual consistency, partial failures, and credential validation without logging secrets.
- Cache/CDN sync changes must not publish draft/private files or stale generated artifacts unintentionally.

## Dash Platform / Storage Rules

- Platform document updates must preserve identities, contract IDs, page slugs, revisions, and provider URLs correctly.
- Slugs and document identifiers have Platform constraints; changes to slug handling or manifest generation need tests for edge cases and collisions.
- Upload/publish flows should be resumable or fail safely. Avoid leaving manifests claiming success when asset uploads or document updates failed.
- Provider-specific code should not leak into provider-agnostic interfaces unless the abstraction is intentionally changing.

## Test Expectations

- New reusable logic in `packages/core`, `packages/scripts`, `packages/gateway`, or DWT server should usually have Vitest coverage.
- New React behavior should have client Vitest/Testing Library coverage when behavior is non-trivial.
- Bug fixes should include a regression test when feasible.
- Cross-platform launcher changes should be reasoned about per OS; tests may be impractical, but reviewers should inspect commands and quoting carefully.

## Things Not To Flag

- The intentional presence of private-repo test credentials in tracked `sites/site-*/.website-config.json` files.
- Missing Electron/Tauri/Rust rewrites; the project is intentionally Node.js + TypeScript.
- Lack of public-hosting hardening unless the PR changes the private/local assumptions or exposes a new boundary.
- Pure generated/static site asset churn unless it affects deployment correctness, links, security, or user-visible behavior.
