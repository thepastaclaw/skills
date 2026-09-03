# Dash Platform Website — Project Skill

## Project Purpose

Private DCG monorepo for the Dash Platform Website / Dash Website Tools (DWT) initiative:

1. Migrate and manage Dash website content through Dash Platform documents plus decentralized asset storage.
2. Build DWT, a locally-run website deployment/management tool that can later be productized for other decentralized site owners.

Repository: `dashpay/dash-platform-website`.

## Boundaries

- This repo is private. Do not quote or expose committed private content outside Dash/DCG contexts.
- Review PRs only. Do not make code changes, branches, commits, or fix PRs against this repo unless explicitly requested.
- Ongoing work is in the `dashpay` repo. Treat any historical `bfoss765/dash-platform-website` copy as context only, not the active repo.

## Tech Stack

- Runtime/language: Node.js 20+ and TypeScript across all packages.
- Monorepo: npm workspaces.
- Core libraries: `dash` JS-SDK, `@lighthouse-web3/sdk`, `@ardrive/turbo-sdk`.
- Backend: Express APIs for DWT and gateway.
- Frontend: React + Vite + TailwindCSS in `packages/deploy-tool/client`.
- Tests: Vitest for root/server/core/scripts/gateway and separate client Vitest run.
- CI: GitHub Actions on Ubuntu/macOS/Windows, Node 20, `npm ci`, `npm run typecheck`, `npm test`.

## Repository Layout

- `packages/core/` — shared wrappers and types: Dash client, Lighthouse, Arweave, storage resolution, manifest handling, DET client, slug handling, atomic filesystem helpers.
- `packages/scripts/` — migration/publishing/extraction scripts for site content and Platform documents.
- `packages/gateway/` — Express DAPI translation layer intended to run behind nginx.
- `packages/deploy-tool/` — DWT Express API plus nested React client workspace.
- `sites/` — git-tracked site content and per-site config/state inputs.
- `docs/` — setup/user guides, architecture references, requirements, design docs.
- `scripts/` — OS-specific DWT launchers.
- `tools/identity-scripts/` — standalone identity creation/top-up scripts.

## Architecture Rules

- `packages/scripts/` should use `@dash-website/core` wrappers; avoid direct SDK calls when a core abstraction exists.
- DET CLI/MCP calls belong behind `packages/core/src/det-client.ts`.
- DWT server routes should keep reusable logic in server/core modules rather than burying complex workflows in route handlers.
- Client code should use the project API wrapper/SSE helpers rather than ad-hoc fetch/event-source code.
- Gateway/CDN/storage changes must preserve clear provider boundaries: Dash Platform documents, decentralized storage provider, gateway/nginx, and Cloudflare/R2/CDN are separate trust and failure domains.
- Prefer cross-platform filesystem/process behavior; CI runs on Windows, macOS, and Ubuntu.

## Sensitive Data Policy for Reviews

`CLAUDE.md` explicitly says Cloudflare API tokens, R2 access keys, and similar credentials are intentionally committed in private `sites/site-*/.website-config.json` files while this repo remains private, to simplify team testing. Do **not** flag their mere presence as a security issue.

Still flag in-scope security bugs when a PR newly broadens exposure, logs/reveals secrets, copies secrets into generated/public artifacts, serves them over APIs without an explicit reveal/confirmation flow, weakens access checks, or changes the agreed private-repo assumption.

## Important Trust Boundaries

- Local DWT browser UI ↔ localhost Express API.
- User-supplied working folder/site content ↔ filesystem operations and publish pipeline.
- Stored config/secrets ↔ reveal endpoints, logs, generated output, and browser-visible data.
- DWT server ↔ SSH gateway management, nginx/pm2 commands, and remote host paths.
- DWT server ↔ Cloudflare/R2 APIs.
- Migration scripts ↔ Dash Platform state transitions and decentralized storage uploads.

## Figma / Design Work Context

The project uses a Figma MCP workflow for design implementation. For design PRs, reviewers should check that assets are downloaded from Figma-provided URLs, SVG `preserveAspectRatio="none"` issues are handled, explicit image dimensions are set where needed, and implementation matches Figma spacing/colors/layout rather than approximating icons or brand assets.
