# AGENTS.md — OP Laughs

Website cataloging One Piece character laughs ("Shishishi", "Zehahaha", "Kuhahaha", …): who laughs, how it's written, and audio clips with episode/chapter sources.

This file covers repo-wide rules. `frontend/AGENTS.md` and `backend/AGENTS.md` cover their subtrees; the closer file wins on conflict.

## Status

Greenfield. Layout below is the **target**; create pieces as needed, following it exactly. Do not invent a parallel structure.

## Repo layout

```
.
├── AGENTS.md
├── Makefile              # the ONLY entrypoint for dev commands
├── compose.yaml          # local dev stack
├── .env.example          # every env var, dev defaults; copy to .env
├── infra/
│   └── dev/              # dev-only config: seaweedfs s3.json, storage-init script
├── backend/              # Go API  → backend/AGENTS.md
│   └── api/openapi.yaml  # API contract, source of truth for both sides
└── frontend/             # React + TS + TanStack SPA → frontend/AGENTS.md
```

## Stack

| Concern | Choice |
|---|---|
| Frontend | React, TypeScript (strict), Vite, TanStack Router (file-based) + TanStack Query, pnpm |
| Backend | Go (stdlib `net/http` routing), pgx/v5, sqlc, goose migrations, `log/slog` |
| API contract | OpenAPI 3 at `backend/api/openapi.yaml`; Go server stubs via oapi-codegen, TS types via openapi-typescript + openapi-fetch |
| Database | PostgreSQL (local: `postgres` image; prod: DigitalOcean Managed PostgreSQL, same major version) |
| Object storage | S3 API only. Local: SeaweedFS (`weed server -s3`). Prod: DigitalOcean Spaces |
| Local runtime | Docker Compose v2 (`docker compose`, never `docker-compose`) driven by `make` |
| Deploy | Docker images → DigitalOcean Container Registry |

The frontend is a pure SPA. No TanStack Start / SSR: the Go service is the only server.

## Local development

### Host prerequisites

Docker (Docker Desktop or OrbStack on macOS; Docker Engine + compose plugin on Linux), GNU make, git. Nothing else. Go, Node, pnpm, sqlc, goose, and air all run inside containers. Host installs of Go/Node are optional and only for editor tooling (gopls, TS server).

### Commands

`make` with no args prints help. Every target is a thin wrapper over `docker compose …`. If a workflow needs a new command, add a Make target; don't document raw `docker` invocations.

| Target | Does |
|---|---|
| `make up` / `make down` | Start / stop the stack (`down` keeps volumes) |
| `make logs [s=backend]` | Follow logs |
| `make sh-backend` / `make sh-frontend` | Shell into a dev container |
| `make psql` | psql into the dev DB |
| `make migrate-up` / `make migrate-down` / `make migrate-new name=add_tags` | goose migrations |
| `make gen` | All codegen: sqlc, oapi-codegen, openapi-typescript |
| `make test` / `make test-backend` / `make test-frontend` | Tests |
| `make lint` / `make fmt` | Lint / format both sides |
| `make fe-add pkg=<name>` | Add a frontend dependency (updates the lockfile as host user) |
| `make reset` | **Destructive:** `down -v`, wipes DB and bucket volumes |
| `make images` | Build production images for `linux/amd64` |

### Dev stack (compose.yaml)

| Service | Port (host) | Notes |
|---|---|---|
| `frontend` | 5173 | Vite dev server with HMR; proxies `/api` → `backend:8080` |
| `backend` | 8080 | air: rebuilds and restarts Go on change |
| `postgres` | `${POSTGRES_HOST_PORT:-5432}` | Named volume `pgdata`; healthcheck gates `backend` |
| `s3` | 8333 | SeaweedFS S3 gateway; creds from `infra/dev/seaweedfs-s3.json` (dev-only) |
| `storage-init` | — | One-shot `amazon/aws-cli`: idempotent bucket create + CORS; `backend` waits on it |

The browser talks only to `localhost:5173` (API via Vite proxy) plus `localhost:8333` for presigned upload/download URLs. Same-origin `/api` in dev mirrors prod, so the backend needs no CORS config for its own API.

### Hot reload

- **Frontend:** source bind-mounted into the container; Vite HMR over port 5173. Native file events by default; set `DEV_WATCH_POLLING=true` in `.env` to switch Vite to polling if edits stop being picked up.
- **Backend:** source bind-mounted; [air](https://github.com/air-verse/air) watches `.go`, `go.mod`, `go.sum`, `.sql`, `.yaml`, rebuilds into `/tmp` inside the container, and restarts the binary (SIGINT first, so graceful shutdown runs). air runs with `poll = true`: across the macOS VM file-sharing boundary, file events depend on the sharing backend and editor save strategy; polling removes that variable at negligible cost for a tree this size. Go module and build caches live in named volumes, so rebuilds after the first are incremental (~1–3 s).

## Cross-platform rules (macOS arm64 ⇄ Linux x86_64)

These are the failure modes that break "works on my other machine". Follow them.

1. **Tooling runs in containers.** No Make target may depend on host Go/Node/pnpm/sqlc. Host tools are for editors only.
2. **Multi-arch images only.** Every image in `compose.yaml` must publish `linux/amd64` and `linux/arm64`. Pin tags; never `latest`. Never set `platform:` in compose for dev services (forces slow emulation on one of the machines).
3. **Never share `node_modules` across OSes.** `frontend/node_modules` in the container is a named volume that shadows the bind mount, so native binaries (esbuild, rollup) always match the container's Linux/arch. A host `pnpm install` for the editor is fine; it never reaches the container.
4. **Use Debian-based Node images, not Alpine.** Avoids musl/glibc optional-dependency mismatches in native packages.
5. **File ownership.** On Linux, a container running as root writes root-owned files into the bind mount; macOS hides this. Dev containers run as `user: "${HOST_UID}:${HOST_GID}"`. The Makefile exports `HOST_UID := $(shell id -u)` / `HOST_GID := $(shell id -g)`. Do **not** name these `UID`/`GID`: `UID` is a read-only bash variable and does not reach `docker compose` reliably. Dev Dockerfiles pre-create mount points for named volumes (`node_modules`, Go caches) with open permissions so fresh volumes inherit them.
6. **GNU make 3.81 compatibility.** macOS ships make 3.81. Don't use `.ONESHELL`, `.RECIPEPREFIX` (3.82+), `!=` assignment, or `$(file …)` (4.0+). Use `:=` and `$(shell …)`.
7. **No BSD/GNU-divergent shell in the Makefile.** No `sed -i`, `readlink -f`, `grep -P`, `date -d`, `xargs -r`. If a recipe needs real scripting, run it inside a container.
8. **Case-sensitive paths.** macOS filesystems are case-insensitive by default, Linux and Docker are not. An import like `./LaughCard` for `laughCard.tsx` passes on the Mac and breaks everywhere else. Match case exactly; never create two paths differing only in case.
9. **Line endings are LF.** Enforced in `.gitattributes` (`* text=auto eol=lf`).
10. **Build for amd64 when deploying.** The Mac is arm64 and DigitalOcean is amd64. `make images` builds with `--platform linux/amd64`. Go Dockerfiles cross-compile via `--platform=$BUILDPLATFORM` + `GOARCH=$TARGETARCH`, not QEMU.

## Configuration

- All runtime config comes from environment variables. `.env.example` lists every variable with a dev default and a one-line comment. `.env` is gitignored. Adding a variable means updating `.env.example` in the same change.
- Never commit real credentials. Dev creds in `infra/dev/` are fake and local-only.
- The same backend image runs locally and in prod; only env differs (S3 endpoint, DB URL, bucket).

## Cross-cutting architecture

- **API contract first.** Change `backend/api/openapi.yaml`, run `make gen`, then implement both sides. Never hand-write request/response types on either side.
- **Generated code** (`*.gen.go`, sqlc output, `frontend/src/api/schema.gen.ts`, `routeTree.gen.ts`) is committed and never hand-edited. CI and `make gen` must be a no-op diff.
- **Media.** Audio and images go to object storage, never Postgres and never the API process's disk. The DB stores object keys, not URLs. Uploads use backend-issued presigned PUT URLs; the browser uploads directly to storage.
- **Domain model (initial):** `characters` (slug, name, epithet, portrait key) → `laughs` (romanized text e.g. "Shishishi", Japanese text e.g. "シシシ", notes) → `clips` (object key, content type, duration, source: anime episode or manga chapter, timestamp). Evolve via migrations only.

## Deployment

- Production images: `backend/Dockerfile` (multi-stage → distroless static, non-root) and `frontend/Dockerfile` (multi-stage → static server with SPA fallback to `index.html`). Dev uses separate `Dockerfile.dev` files; don't merge them.
- Pushed to DigitalOcean Container Registry. DB: DO Managed PostgreSQL. Storage: DO Spaces.
- Migrations run as an explicit release step (`api migrate up` from the backend image), never implicitly on server start.
- **Undecided:** App Platform vs Droplet + compose, and how `/api` is routed in prod. Ask before adding deploy manifests, CI/CD, or a reverse proxy.

## Open decisions (ask before choosing)

- Styling approach / component library
- Auth for contributors/admins (the public catalog is read-only)
- Deploy runtime (see above)

## Agent working rules

- Run everything through `make`. If the stack isn't up, `make up` first.
- Verify changes in the running stack (hit the endpoint, load the page), not just via tests.
- Don't run `make reset` or delete volumes without asking.
- A merged migration is immutable; fix forward with a new one.
- Keep this file and the sub-`AGENTS.md` files current when you change layout, commands, or conventions.
