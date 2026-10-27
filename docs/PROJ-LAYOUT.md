# Project Layout

Map of the `noizu-forks/livebook` checkout. This is a **fork of upstream
Livebook** (`livebook-dev/livebook`) — an Elixir/Phoenix web application for
interactive code notebooks. The tree at the checked-out SHA carries **no
Noizu-specific source divergence**: every commit in history is an upstream
author's, and no `noizu` references exist in the code. Treat the layout below
as upstream Livebook's, with deployment context in
[PROJ-ARCH.md](PROJ-ARCH.md) describing how Noizu runs it.

```
livebook/
├── lib/                          # Application source (see layout note below)
│   ├── livebook.ex               # Public Elixir API + config_runtime/0 (env plumbing)
│   ├── livebook/                 # Domain core  (see lib/ breakdown below)
│   ├── livebook_web/             # Phoenix web layer  (see lib/ breakdown below)
│   ├── livebook_cli/             # `livebook` CLI (server / deploy / etc.)
│   └── mix/tasks/                # Mix tasks (livebook.gen.*, etc.)
├── config/                       # Config files
│   ├── config.exs                # Base config (endpoint, auth default, apps manager)
│   ├── dev.exs / test.exs        # Env overrides
│   ├── prod.exs                  # Bind 127.0.0.1:8080, iframe 8081, warning logs
│   └── runtime.exs               # Calls Livebook.config_runtime/0 (reads LIVEBOOK_* env)
├── rel/                          # Release assets
│   ├── server/                   # OTP release env/vm.args templates
│   └── app/                      # Desktop app (Tauri, standalone.esx) — Rust shell
├── proto/                        # Protobuf messages for Livebook Teams hub sync
├── iframe/                       # Separate Mix project served on the iframe port
├── assets/                       # Frontend (bun + vite): css/, js/, public/, test/
├── docs/                         # Upstream docs (authentication/, deployment/, teams/) + this doc set
├── test/                         # ExUnit test suite
├── priv/                         # Static/generated runtime files (.gitkeep at rest)
├── .github/workflows/            # CI (ci.yml) + release (release.yml)
├── Dockerfile                    # Multi-stage build (default + CUDA variants), release image
├── mix.exs                       # Project definition, deps, releases, aliases
├── mix.lock                      # Locked dependency versions
├── versions                      # Pinned elixir / otp / rebar3 / ubuntu versions
├── README.md                     # Upstream entry point + env-var reference
├── RELEASE.md                    # Upstream release runbook (hex.pm publish)
├── SECURITY.md                   # Upstream vulnerability disclosure policy
├── CHANGELOG.md                  # Keep-a-Changelog, upstream v0.20
├── LICENSE                       # Apache-2.0
├── .editorconfig                 # Editor settings
├── .formatter.exs                # mix format inputs
├── .dockerignore                 # Container build context excludes
└── .gitattributes                # Git attributes
```

> `layout/lib.md` does not exist yet — the `lib/` breakdown below covers it
> inline until it merits extraction.

## lib/ breakdown

| Path | Contents |
|------|----------|
| `lib/livebook/application.ex` | OTP entry: supervision tree, storage, endpoint, runtimes, ZTA |
| `lib/livebook/storage.ex` | ETS dump-file entity-attribute-value store |
| `lib/livebook/apps/` | Permanent app deployments (manager, deployer, app specs) |
| `lib/livebook/runtime/` | Notebook evaluators: standalone, attached, Fly, K8s, embedded |
| `lib/livebook/session/` | Notebook session process + data structures |
| `lib/livebook/notebook/` | Notebook sections/cells and `.livemd` parsing |
| `lib/livebook/hubs/` | Hub providers (personal filesystem, Livebook Teams) + sync client |
| `lib/livebook/secrets/` | Secret struct; hub-backed storage |
| `lib/livebook/teams/` | Livebook Teams API requests |
| `lib/livebook/zta/` | Zero-trust identity: `LivebookTeams` provider |
| `lib/livebook/k8s/` | Kubernetes runtime pod/PVC manifests |
| `lib/livebook/proxy/` | Runtime-side request proxying (custom plug adapter) |
| `lib/livebook/file_system/` | Filesystem abstractions (local, S3) |
| `lib/livebook/live_markdown/` | Markdown → LiveView rendering |
| `lib/livebook/intellisense/`, `text/` | Completion, formatting, code helpers |
| `lib/livebook_web/router.ex` | HTTP + LiveView routes, auth pipelines |
| `lib/livebook_web/endpoint.ex` | Phoenix endpoint, sockets, gzip static, force_ssl |
| `lib/livebook_web/live/` | LiveView pages (home, session, hub, app, settings, …) |
| `lib/livebook_web/plugs/` | `auth_plug`, `user_plug`, `proxy_plug`, `configured_plug` |
| `lib/livebook_web/channels/` | `Socket` + `js_view_channel` (JS widget bridge) |

## Key files requiring setup

| File | Action |
|------|--------|
| `versions` | Pinned toolchain; bump only with rebar3 compatibility |
| Environment | Configure via `LIVEBOOK_*` env vars (read in `Livebook.config_runtime/0`); see README |
| `LIVEBOOK_PASSWORD` / `LIVEBOOK_SECRET_KEY_BASE` | Required for a protected instance in this deployment |

Full directory contents: `docs/` here mirrors upstream reference material; see
[PROJ-ARCH.md](PROJ-ARCH.md) for how the running instance is deployed.
