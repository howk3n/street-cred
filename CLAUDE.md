# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## Project

<!-- TODO: One paragraph — what street-cred is, who uses it, core flows. Auth + data providers. -->

## Layout

<!-- TODO: Fill in once scaffolded. -->

```
apps/
packages/
supabase/
docs/superpowers/
  specs/        Design + decision docs
  plans/        Phase plans
  reviews/      Code reviews
```

## Commands

<!-- TODO: Fill in once tooling is chosen. -->

```bash
pnpm install
pnpm dev
pnpm build
pnpm lint
pnpm test
```

## Architecture

<!-- TODO: Auth contract, data layer, app wiring, backend functions, styling, path aliases. -->

## Conventions

- **Explicit save.** All DB writes need an explicit save/confirm button. No auto-save on blur, debounce, or implicit triggers.
- **Hooks gate data access.** Components never call the DB client directly — extract to `api.ts` + a hook in `queries.ts`, even for one-off calls.
- **Query keys via factories.** Every domain has a `*Keys` object; invalidations import the factory rather than typing the array inline.
- **Never trust caller IDs.** Server-side code (RPCs, functions) re-derives caller identity from the auth token, never from an `_id` parameter.
- **No comments unless the WHY is non-obvious.** Prefer naming. Pull `// Used by X` and `// Added for Y` out of code — those belong in PR descriptions.

## Phase status

<!-- TODO -->
