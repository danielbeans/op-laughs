# AGENTS.md — frontend (React + TypeScript + TanStack)

Repo-wide rules live in `../AGENTS.md`. Run commands through the root Makefile (`make test-frontend`, `make fe-add pkg=…`, `make sh-frontend`); don't assume host Node.

## Stack

- React + TypeScript (`strict`, `noUncheckedIndexedAccess`), Vite.
- TanStack Router, file-based via `@tanstack/router-plugin/vite`; generates `src/routeTree.gen.ts` (committed, do not edit).
- TanStack Query for all server state.
- API client: `openapi-fetch` typed by `openapi-typescript` output from `../backend/api/openapi.yaml` → `src/api/schema.gen.ts` (do not edit; regenerate with `make gen`).
- Tests: Vitest + Testing Library. Lint/format: ESLint (flat config, typescript-eslint) + Prettier.
- pnpm; version pinned in `package.json` `packageManager` and installed in `Dockerfile.dev` to match.
- Styling: **not chosen yet**. Ask before adding a CSS framework or component library.

## Layout

```
frontend/
├── src/
│   ├── main.tsx            # router + QueryClient providers
│   ├── routeTree.gen.ts    # generated
│   ├── routes/             # file-based routes (TanStack Router conventions)
│   │   ├── __root.tsx
│   │   ├── index.tsx                   # browse / search laughs
│   │   ├── characters/index.tsx
│   │   └── characters/$slug.tsx        # character page with their laughs + clips
│   ├── api/
│   │   ├── schema.gen.ts   # generated
│   │   ├── client.ts       # single openapi-fetch client, baseUrl "/api/v1"
│   │   └── queries/        # queryOptions factories per resource
│   ├── features/           # feature-scoped components/hooks (laughs, characters, upload, player)
│   └── components/         # shared, domain-agnostic UI
├── vite.config.ts
├── Dockerfile              # prod: build → static server with SPA fallback
└── Dockerfile.dev          # dev: node (Debian), Vite dev server, runs as HOST_UID:HOST_GID
```

## Dev server (container)

`vite.config.ts` must keep:

- `server.host: true`, `server.port: 5173`, `server.strictPort: true`, so it's reachable through the compose port mapping and HMR's websocket port stays predictable.
- `server.proxy['/api'] → 'http://backend:8080'` (compose service name). Code always calls relative `/api/v1/...`. Never hardcode `localhost:8080` or an absolute API origin.
- `server.watch.usePolling` driven by `process.env.DEV_WATCH_POLLING === 'true'` (fallback for missed file events on macOS).

`node_modules` inside the container is a named volume. Add deps only with `make fe-add pkg=…` so the lockfile is written as the host user and the container volume is updated. A host `pnpm install` for editor tooling is fine but never used by the container.

`../backend/api/openapi.yaml` is mounted read-only into the frontend container for codegen.

## Data fetching

- Every query is defined once as a `queryOptions(...)` factory in `src/api/queries/`. Components and loaders import the factory; no inline query keys.
- Query keys are hierarchical arrays: `['characters']`, `['characters', slug]`, `['laughs', { characterSlug, q }]`.
- Route loaders prefetch with `queryClient.ensureQueryData(factory(...))`; components read with `useSuspenseQuery(factory(...))`. Router context carries the `QueryClient`.
- Search/filter state lives in the URL via route `validateSearch` (typed), not in component state, so views are linkable.
- Mutations invalidate the narrowest affected key.
- Never `fetch` directly; always the typed client.

## Media

- Audio playback: one shared player (feature `player`) so only one laugh plays at a time; native `<audio>` element. Clip URLs come from the API (presigned or public); never construct storage URLs client-side.
- Upload: request presigned PUT from API → `PUT` the file directly to the returned URL with the exact `Content-Type` that was signed → call the API confirm endpoint. Validate type/size client-side before requesting, but the backend is authoritative.

## Conventions

- Import paths must match file case exactly (Linux/Docker are case-sensitive; macOS is not).
- Components: PascalCase files. Hooks: `useX.ts`. No default exports except where TanStack Router file routes require `Route` exports.
- No `any`; no non-null `!` on API data. Handle the optional fields the schema declares.
- Accessibility: play buttons have labels naming the character and laugh; keyboard-operable player.

## Testing

- `make test-frontend` runs Vitest in the container.
- Test user-visible behavior (renders the laugh list for a character, search params filter results, upload shows an error on rejected type). Mock at the network boundary (MSW or a stubbed openapi-fetch client), not TanStack internals.
