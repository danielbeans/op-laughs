# AGENTS.md — backend (Go API)

Repo-wide rules live in `../AGENTS.md`. Run commands through the root Makefile (`make test-backend`, `make gen`, `make sh-backend`); don't assume host Go.

## Layout

```
backend/
├── cmd/api/main.go         # entrypoint; subcommands: `serve` (default), `migrate up|down|status`
├── api/
│   ├── openapi.yaml        # API contract (source of truth)
│   └── oapi-codegen.yaml   # codegen config
├── internal/
│   ├── config/             # env → typed Config; fail fast on missing/invalid
│   ├── httpapi/            # handlers implementing the generated strict server interface
│   │   └── api.gen.go      # generated, do not edit
│   ├── catalog/            # domain logic: characters, laughs, clips
│   ├── db/
│   │   ├── migrations/     # goose SQL migrations, embedded via embed.FS
│   │   ├── queries/        # sqlc input (*.sql)
│   │   └── sqlc/           # sqlc output, do not edit
│   └── storage/            # S3 client wrapper: presign PUT/GET, head, delete
├── sqlc.yaml
├── .air.toml
├── Dockerfile              # prod: multi-stage → gcr.io/distroless/static-debian12:nonroot
└── Dockerfile.dev          # dev: golang image + air, runs as HOST_UID:HOST_GID
```

Dependency direction: `httpapi` → `catalog` → `db/sqlc`, `storage`. `catalog` never imports `httpapi` or generated HTTP types; map at the handler boundary.

## Hot reload (air)

`.air.toml` essentials, which must stay this way:

- `cmd = "go build -o /tmp/bin/api ./cmd/api"`, `bin = "/tmp/bin/api"`: build output stays outside the bind mount (no root-owned or stray artifacts in the repo).
- `tmp_dir` is gitignored if it lands in-tree.
- `poll = true`: required for reliable reloads through macOS Docker file sharing (see root AGENTS.md).
- `include_ext = ["go", "sql", "yaml"]`; exclude `*_test.go` from triggering restarts.
- `send_interrupt = true` with a short `kill_delay`, so the server's graceful shutdown path runs on every reload.

`GOMODCACHE` and `GOCACHE` point at named volumes. Don't `go mod vendor`.

## HTTP

- Stdlib `net/http` + `http.ServeMux` method/path patterns. No web framework.
- Handlers implement the oapi-codegen **strict** server interface (`std-http-server`). To add an endpoint, edit `api/openapi.yaml`, run `make gen`, then implement the new interface method. The compiler tells you what's missing.
- All API routes live under `/api/v1`. Unversioned: `GET /healthz` (process up) and `GET /readyz` (DB ping + storage reachable).
- Errors: one JSON error shape defined in the OpenAPI spec. Domain errors (`catalog.ErrNotFound`, validation errors) map to status codes in one place in `httpapi`. Never leak SQL or S3 error text to clients; log it.
- `context.Context` flows from the request into every DB and storage call. No `context.Background()` below `main`.
- Server has explicit read/write/idle timeouts and shuts down gracefully on SIGINT/SIGTERM.
- Logging: `log/slog` JSON handler, one request log line (method, route, status, duration, request id). Log level via env.

## Database

- pgx/v5 pool (`pgxpool`), created once in `main` and injected. sqlc with `sql_package: "pgx/v5"`.
- New query: add SQL to `internal/db/queries/*.sql`, run `make gen`, and call the generated method. No hand-built SQL strings, no ORM.
- Migrations: `make migrate-new name=…` creates a goose SQL file. Each migration has a working `-- +goose Down`. Merged migrations are immutable.
- Migrations are embedded in the binary and run via `api migrate up`: `make migrate-up` locally, a release step in prod. The server never auto-migrates on start.
- IDs: UUIDs (`uuid` column, generated in Postgres). Timestamps: `timestamptz`, UTC.
- Human-facing identifiers (character slugs like `monkey-d-luffy`) are unique, indexed, and used in URLs. UUIDs stay internal to the API.
- Multi-step writes (e.g. create laugh + clip rows) run in one transaction via sqlc's `WithTx`.

## Object storage

- aws-sdk-go-v2 `s3` client, configured from env: `S3_ENDPOINT`, `S3_PUBLIC_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_USE_PATH_STYLE`.
  - Local: endpoint `http://s3:8333`, path-style `true`. Prod (Spaces): `https://<region>.digitaloceanspaces.com`, path-style `false`.
- Set `RequestChecksumCalculation` and `ResponseChecksumValidation` to `WhenRequired`. The SDK's default checksum behavior is not supported by all S3-compatible stores.
- **Two clients:** internal calls (head/delete) use `S3_ENDPOINT`. Presigned URLs must be signed against `S3_PUBLIC_ENDPOINT` (`http://localhost:8333` in dev), because the host is part of the signature and the browser can't resolve `s3`.
- Upload flow: client requests upload → backend validates content type/size and returns presigned PUT (short expiry) + object key → browser PUTs directly → client confirms → backend `HeadObject`s to verify, then writes the `clips` row.
- Object keys: `clips/<laugh-id>/<clip-id>.<ext>`, `portraits/<character-id>.<ext>`. Never derived from user-supplied filenames.
- Allowed audio types: `audio/mpeg`, `audio/ogg`, `audio/webm`, `audio/mp4`. Enforce a max size in config.

## Testing

- `make test-backend` runs `go test ./...` in the dev container against the compose Postgres, using a separate test database created and migrated by the test harness.
- Domain and DB behavior: integration tests against real Postgres. Don't mock sqlc.
- Handlers: `httptest` through the real mux + strict server wiring.
- Storage: test against the compose SeaweedFS bucket under a test key prefix; clean up after.
- Table-driven tests. `t.Parallel()` where there's no shared DB state.

## Style

- `gofmt`/`goimports` clean; `golangci-lint` via `make lint`.
- Wrap errors with context: `fmt.Errorf("get character %q: %w", slug, err)`.
- No package-level mutable state; dependencies are passed via structs built in `main`.
- No `panic` outside `main` init.
