# RepoGraph — Codebase Context

## What this project is

RepoGraph is a hosted codebase-analysis application. A user imports a Java,
JavaScript, TypeScript, or Python repository; a Python background worker parses it
and stores a file/import inventory; a SvelteKit workspace presents six visual
analyses and an optional source-grounded chat. Source code is uploaded to and
processed by this application for imports (it is not a browser-only/local-only
analyzer).

The repository has two cooperating runtimes:

1. A Node.js/SvelteKit web application for the UI, HTTP routes, GitHub OAuth,
   PostgreSQL access, graph analysis, and chat.
2. A long-running Python process that claims PostgreSQL parse jobs, fetches or
   unpacks source, runs Tree-sitter extraction, and persists results.

## Architecture and data flow

```text
Browser (Svelte 5 / D3)
  ├─ folder or ZIP ── filter + ZIP ──┐
  ├─ public GitHub reference ────────┼─ POST /api/projects
  └─ sample project ─────────────────┘
                                        │
                           PostgreSQL transaction
                           projects + parse_jobs
                           (uploads also in project_uploads)
                                        │
                           Python worker_loop.py
                           clone/extract → inventory
                           Tree-sitter → resolve imports
                                        │
                  transaction: files, source, edges, types,
                  entities, project ready + parsed_at
                                        │
                           SvelteKit workspace
                           graph/types/entities APIs
                           + pure TS analysis package
                                        │
                         six interactive views + chat
```

### Import and parse lifecycle

- The landing page is `src/routes/+page.svelte`; `src/lib/import/client.ts`
  collects folder/drop files, applies a browser-side filter, zips folder imports,
  and uploads with progress. ZIP files can also be selected or dropped.
- `src/routes/api/projects/+server.ts` accepts:
  - JSON `{ "github": "owner/repo", "branch": "optional" }` for a public GitHub
    repository;
  - JSON `{ "sample": true }` for the bundled fixture;
  - multipart form data with an `archive` ZIP for local uploads.
- Project creation, optional archive persistence, and queue insertion happen in
  one database transaction. Uploads require a signed-in user. GitHub and sample
  imports can be anonymous; anonymous projects have no owner and are accessible
  to anyone with the project link. Projects created while signed in are owner-only.
- `workers/worker_loop.py` claims `parse_jobs` with row locking and
  `SKIP LOCKED`, retries transient failures, reclaims stale jobs, and prunes
  completed/failed job records. Run it as its own supervised process, not as a
  per-request child of the web app.
- `workers/source.py` shallow-clones public GitHub repositories or securely
  extracts ZIPs. Import size/file-count caps are enforced on fetched/extracted
  content. ZIP paths are checked against traversal/absolute paths; symlinks are
  not materialized. Temporary work directories are removed after processing.
- `workers/filters.py` is the authoritative language/path filter. The matching
  browser copy is `src/lib/import/filters.ts`; parity is tested using
  `tests/fixtures/filter-cases.json`. Supported extensions are Java, JS/JSX/MJS/CJS,
  TS/TSX/MTS/CTS, and Python. Generated/declaration files, ignored directories,
  and files over 1.5 MB are excluded or counted as skipped.
- `workers/inventory.py`, `workers/extract.py`, and `workers/entities.py` use
  Tree-sitter grammars to extract non-blank LOC, imports, class/interface
  declarations and ORM entities. `workers/resolve.py` resolves statically
  identifiable intra-repository Java, JS/TS, and Python imports to file edges;
  external dependencies do not become edges. Dynamic or non-literal imports may
  not resolve.
- All parsed rows and the successful job/project status transition are committed
  together. Retried parses clear earlier parsed rows first. Only parsed source
  contents persist in PostgreSQL; working copies are temporary.
- The workspace polls `/api/projects/:id/status` while an import is pending,
  cloning, or parsing, then loads the graph. Types and entities are loaded lazily
  when needed.

### Graph analysis and UI

- `packages/analysis/graph/` is a framework- and database-independent TypeScript
  package. It derives adjacency/reverse adjacency, fan-in/out, iterative Tarjan
  strongly connected components, BFS reachability, compressed path trees,
  grouping levels, and default grouping.
- `packages/analysis/hotspots/` implements ranked hotspot lists, import cycles,
  and the "possibly unused" heuristic. The latter excludes tests and recognized
  entry points and cross-references `extends`/`implements` declarations.
- `packages/analysis/types/` resolves class/interface supertypes and ORM
  relationships within language/ORM scopes, avoiding ambiguous name matches.
- The graph API returns the full unfiltered graph. `src/lib/workspace/state.svelte.ts`
  owns the client workspace state and re-derives filtered graph/tree data as
  language, hide-tests, and search controls change. Chat and cross-file type
  resolution use the full graph, not the current visual filters.
- The six views are Tree, Size map (treemap), Dependencies (force graph),
  Architecture (components/classes), ER Diagram, and Hotspots. Their Svelte
  components and view-specific rendering/layout helpers are in
  `src/lib/views/`; D3 owns complex SVG/force-simulation rendering where needed.
- `src/lib/workspace/` contains shared workspace state, toolbar, side panel,
  chat panel/settings, filters, and navigation helpers. `src/routes/workspace/`
  assembles the view and handles import progress/error states.

### Authentication, persistence, and HTTP surfaces

- GitHub OAuth routes are under `src/routes/auth/github/`; state and PKCE
  verification are supported. `src/hooks.server.ts` loads the session user into
  `event.locals`.
- `src/lib/server/auth/guards.ts` centralizes signed-in-user and project access
  checks. Owned projects are hidden as 404 from other users; ownerless public
  projects are link-accessible.
- PostgreSQL access uses `postgres` (Postgres.js). Routes should call named
  repository functions in `src/lib/server/db/`, not issue SQL directly.
  `src/lib/server/db/transaction.ts` owns transaction typing/wiring.
- `db/migrate.ts` applies sorted `db/migrations/*.sql` files once, each in its
  own transaction. The schema covers users/sessions/OAuth state, projects,
  uploads, parse jobs, parsed files/edges/types/entities, and saved chat keys.
- Main routes:
  - `POST /api/projects`: create/queue import.
  - `GET /api/projects/:id/status`: status and summary.
  - `GET /api/projects/:id/graph`: graph, derived metrics/tree/group levels.
  - `GET /api/projects/:id/types` and `/entities`: parser metadata.
  - `POST /api/projects/:id/chat`: chat against an accessible, ready project.
  - `/api/chat/keys` (`GET`/`POST`/`PATCH`/`DELETE`) and
    `/api/chat/models` (`POST`): manage BYO model keys and model discovery.
  - `/projects`: signed-in user's project list and delete action.
  - `/workspace/:projectId`: project workspace.
- `src/lib/server/graphCache.ts` provides an in-memory, per-process LRU cache for
  derived graph/types/entities/chat data. Entries are versioned by `parsed_at`;
  it is not a shared cache across app instances.

### Chat

- Chat requires a signed-in user and the user's own API key. Supported providers
  are OpenAI, Anthropic (Claude), Google Gemini, and xAI (Grok). Provider-neutral
  request/tool types and adapters are in `src/lib/server/chat/`.
- API keys are encrypted at rest with AES-256-GCM using `CHAT_KEY_SECRET`. The
  browser only receives provider/model metadata and a last-four-character hint.
  The saved model is probed before use.
- `src/lib/server/chat/converse.ts` builds a repository context and runs a bounded
  tool-call loop. The tool registry in `src/lib/server/chat/tools.ts` exposes
  `getFile`, `searchFiles`, and `listChildren`. File source reads go through
  `src/lib/server/fileStorage.ts`.
- Chat request history is capped at 40 visible turns and 8,000 characters per
  turn; tool execution is capped at 8 model rounds and model output at 4,096
  tokens. The conversation is held in workspace state, not persisted.
- There is no application-level daily chat quota or Stripe billing implementation
  in the current migrations/routes. Chat usage is billed to the user's configured
  provider key.

## Repository map

| Path | Responsibility |
|---|---|
| `src/routes/` | SvelteKit pages, OAuth endpoints, project APIs, chat APIs |
| `src/lib/import/` | Browser-side import helpers, GitHub reference validation, filter parity |
| `src/lib/server/auth/` | OAuth/PKCE, cookies, access guards |
| `src/lib/server/db/` | PostgreSQL repositories for users, projects, parsed data, sessions, chat keys |
| `src/lib/server/chat/` | Context construction, tools, provider adapters, key encryption/settings |
| `src/lib/workspace/` | Workspace state and shared controls/panels |
| `src/lib/views/` | Six visualizations, their models, and layout algorithms |
| `packages/analysis/` | Pure graph, hotspot, type/entity resolution logic and Vitest specs |
| `workers/` | Python import worker, Tree-sitter extraction, resolution, filtering, storage |
| `db/migrations/` | Incremental PostgreSQL schema |
| `tests/unit/` | Python worker/filter/parser/resolver unit tests |
| `tests/integration/` | Python worker/database integration tests |
| `fixtures/sample-project/` | Multi-language sample codebase imported through the normal pipeline |
| `docs/repograph-spec.md` | Detailed specification of the earlier single-file browser prototype |
| `BUILD_PLAN.md` | Historical phased build plan; useful background, not current status |

## Development and validation

### Prerequisites

- Node.js `>=22.18` (see `package.json`).
- Python `>=3.11` and `uv` (see `.python-version`, `pyproject.toml`, and `uv.lock`).
- PostgreSQL; `docker-compose.yml` provides PostgreSQL 16 for local development.
- Git for public GitHub imports.

### Typical local startup

1. Install JS dependencies: `npm install`.
2. Copy `.env.example` to `.env` and configure `DATABASE_URL`.
3. Start the database: `docker compose up -d`.
4. Apply migrations: `npm run db:migrate`.
5. Start the SvelteKit app: `npm run dev`.
6. In a separate terminal, start the parser: `npm run worker`.

OAuth requires a configured GitHub OAuth app. Chat requires `CHAT_KEY_SECRET`,
then a signed-in user must add a supported provider key through chat settings.
The app's env definitions are in `src/env.ts`; the worker reads its settings from
the environment in `workers/worker_loop.py`.

### Useful commands

| Command | Purpose |
|---|---|
| `npm run check` | SvelteKit sync and Svelte/TypeScript checks |
| `npm run lint` | Prettier check followed by ESLint |
| `npm run test:unit` | Vitest specs under `src/` and `packages/analysis/` |
| `npm run test:py` | Python unit tests |
| `npm run test:py:integration` | Python integration tests (configured database/environment required) |
| `npm run build` | Production SvelteKit build |
| `npm run db:migrate` | Apply outstanding SQL migrations |
| `npm run worker` | Run the long-lived parse worker |
| `npm run worker:watchdog` | Reclaim stale jobs without processing them |

## Configuration and operational notes

The current environment template is `.env.example`. Important variables:

- `DATABASE_URL`: shared PostgreSQL connection string for app and worker.
- `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `GITHUB_CALLBACK_URL`: OAuth login.
- `CHAT_KEY_SECRET`: base64-encoded 32 random bytes for encrypting saved API keys;
  changing it makes existing saved keys unreadable.
- `IMPORT_MAX_BYTES` (default 200 MiB) and `IMPORT_MAX_FILES` (default 20,000):
  import caps.
- `PARSE_MAX_ATTEMPTS` (default 3) and `PARSE_STALE_TIMEOUT_MIN` (default 15):
  worker retry and stale-claim behavior.
- `BODY_SIZE_LIMIT` may need raising for production ZIP uploads; the template
  documents a suitable value.
- Stripe settings appear as optional template placeholders but have no current
  billing flow.

The app validates configured import limits as positive integers and validates
`CHAT_KEY_SECRET` length/encoding in `src/env.ts`. Most settings are optional at
startup; the feature that needs a missing setting reports an error when used.

## Important implementation constraints

- Keep graph-analysis package code pure: no SvelteKit, database, or DOM imports.
- Keep SQL behind `src/lib/server/db/` repository functions. Use
  `withTransaction` when an operation must be atomic.
- Keep TypeScript and Python file-filter rules in parity; update their shared
  fixture/tests when changing supported paths or extensions.
- Preserve parse-job atomicity and claim checks. A stale attempt must not overwrite
  results from a newer claim; temporary clones/extractions must be cleaned up.
- Keep source-content access behind `src/lib/server/fileStorage.ts` on the app
  side and `workers/file_storage.py` on the worker side.
- All project-scoped endpoints/pages should call `requireProjectAccess` before
  reading project data. Public projects are intentionally link-accessible;
  owned projects are private.
- The graph returned to the browser/chat is the full graph. Apply language/test
  filters to view derivation, not to stored data or chat tools.
- `parsed_at` is the cache version. Re-imports should update it in the same
  transaction that publishes the new parsed state.

## Documentation drift / source of truth

The checked-in implementation is materially newer than the original planning
documents:

- `README.md` says the repository does not exist yet, describes the old planned
  phases, and contains outdated prerequisites, tests, and Anthropic/server-side
  quota assumptions.
- `BUILD_PLAN.md` and `docs/repograph-spec.md` describe intended or prototype
  behavior. The spec's original app was a single HTML file with local-only
  parsing; the current product is SvelteKit + PostgreSQL + a Python worker.
- The current app supports public GitHub imports and authenticated ZIP uploads;
  the worker deliberately clones GitHub without credentials, so private GitHub
  repositories are not supported by that path.
- Current chat is bring-your-own-key for four providers, with encrypted saved
  keys; it is not limited to a server-owned Anthropic key. Planned quotas,
  billing, account/usage pages, and Stripe integration are not implemented.
- The package scripts include Python worker/unit/integration tests, but do not
  define the README's claimed `test:integration` or `test:e2e` commands. The
  repository currently has no Playwright/e2e setup in `package.json`.

For code behavior, prefer the implementation and tests over those historical
descriptions. Update the appropriate planning docs separately if a future task
requests documentation reconciliation.