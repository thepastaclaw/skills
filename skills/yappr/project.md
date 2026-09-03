# Yappr — Project Skill

## Project Purpose

`PastaPastaPasta/yappr` is a public Next.js social app and marketplace built on Dash Platform. It stores posts, replies, likes, follows, profiles, private feeds, encrypted messages, storefront data, orders, tips, bookmarks, and moderation-related documents through Dash Platform contracts.

Repository: `PastaPastaPasta/yappr`.

## Boundaries

- This repo is public. Do not expose private Dash/DCG context in GitHub review text.
- Review PRs only. Do not push code changes, branches, commits, or fix PRs unless explicitly requested.
- Treat user keys, WIFs, mnemonics, identity private keys, encrypted backup material, Pinata/Storacha credentials, and payment/address data as sensitive.

## Tech Stack

- Framework: Next.js 14 App Router with static export support.
- Language: TypeScript and React 18.
- Platform integration: `@dashevo/evo-sdk`.
- Styling/UI: Tailwind CSS, Radix UI, Headless UI, Framer Motion, Heroicons, Lucide.
- State: React contexts, hooks, and Zustand.
- Package manager: npm with `package-lock.json`.

## Repository Layout

- `app/` — Next.js route pages.
- `components/` — UI, auth, compose, post, settings, profile, feed, storefront, and token components.
- `contexts/` — auth and SDK providers.
- `hooks/` — React hooks for auth, feeds, posts, follows, validation, uploads, private feeds, and modals.
- `lib/services/` — Dash Platform service layer for documents, identities, posts, likes, replies, follows, reposts, profiles, stores, messages, tokens, and state transitions.
- `contracts/` — Dash Platform data contracts.
- `types/` — shared app/domain types.
- `scripts/` and root JS helpers — contract/token/setup utilities.

## Architecture Notes

- Prefer central service-layer fixes in `lib/services/` over duplicating SDK calls in UI components.
- Write operations should pass through `state-transition-service` or the existing service abstraction when possible.
- Platform query/index changes must match the active data contract indexes and the app's query shapes.
- Token/payment changes need careful handling of credits, BigInt/precision, max-cost guards, balance checks, and readable failure states.
- Private feed, encrypted backup, DM, upload-provider, and checkout paths cross sensitive-data boundaries; review them for accidental persistence, logging, or disclosure.
