# Project Schema

## Scope

Livebook has **no relational database** and no Liquibase/Ecto migrations
against a SQL store — there is nothing to draw a SQL ERD for. `Ecto` is used
only for `embedded_schema` structs and changesets (validation), not
persistence. Structured data in this repo is:

1. a **keyspace** (`Livebook.Storage`): an ETS table persisted to a single ETS
   dump file, holding all configuration/state entities;
2. **filesystem data**: notebooks (`.livemd`), app data, secrets, filesystem
   credentials, dependency caches;
3. **config schema**: `LIVEBOOK_*` environment variables read at boot;
4. **interface schemas**: HTTP routes, LiveView/JS sockets, the iframe
   endpoint, dev endpoints, and protobuf messages for Livebook Teams sync.

No ERD sections follow; instead each data kind is documented below.

## Persistence — Livebook.Storage (EAV)

`Livebook.Storage` (`lib/livebook/storage.ex`) is an
entity-attribute-value store. Reads hit an ETS table directly; writes go
through a GenServer that also persists the table to disk.

**Dump file**: `"#{data_path}/livebook_config.v#{migration_version}.ets"`
(an ETS table dump, not JSON). The `migration_version` in the filename lets a
schema change load a fresh store; `Livebook.Migration.run/0` migrates old
versions at boot.

**Record shape** (conceptual): `{{namespace, id}, attribute, value, timestamp}`
where `id` is a binary and each attribute is an atom key.

| Namespace | id | Attributes (examples) | Written / read by |
|-----------|----|-----------------------|-------------------|
| `:system` | `"global"` | `migration_version` | `Livebook.Migration` |
| `:settings` | `"global"` | `autosave_path`, `update_check_enabled`, `dev_endpoints_enabled`, … | `Livebook.Settings` |
| `:env_vars` | env var name | `name`, `value` | `Livebook.Settings` |
| `:hubs` | hub id | hub fields (`hub_name`, `hub_emoji`, …) | `Livebook.Hubs` |
| `:default_hub` | `"default_hub"` | `default_hub` (hub id) | `Livebook.Hubs` |
| `:secrets` | secret name | secret refs | `Livebook.Hubs` / `Secrets` |
| `:filesystem` | filesystem id | `type`, `bucket_url`, `secret`, `access_key`, … | `Livebook.FileSystems` |
| `:notebook_manager` | `"global"` | notebook manager attrs | `Livebook.NotebookManager` |

The personal hub keeps its own sub-keyspaces (`:hub_secrets`,
`:file_systems`) via `Livebook.Hubs.Personal`. **Secrets are stored encrypted**
with the hub's `secret_key` (64 bytes) and marked `redact: true`.

## Filesystem data layout

| Path | Env / default | Contents |
|------|---------------|----------|
| Home | `LIVEBOOK_HOME` / user home | Working directory for notebooks |
| Data path | `LIVEBOOK_DATA_PATH` / OS user-data dir | `livebook_config.v*.ets` dump; app data |
| Apps path | `LIVEBOOK_APPS_PATH` | Optional auto-deployed apps from a folder |
| Mix installs | `MIX_INSTALL_DIR` | Cached `Mix.install/2` deps per notebook |
| Temp | system tmp `/livebook/<version>` | Runtime temp files |

In the deployed instance (see [PROJ-ARCH.md](PROJ-ARCH.md)) these map under
`/data/{home,notebooks,app-data,.mix-installs}` on a 10Gi PVC.

## Secrets

`Livebook.Secrets.Secret` (`lib/livebook/secrets/secret.ex`) is an
`embedded_schema`:

| Field | Type | Rules |
|-------|------|-------|
| `name` | string (PK) | upper-cased on write; `^\w+$` (alphanumeric + underscore) |
| `value` | string | required |
| `hub_id` | string | owning hub |
| `deployment_group_id` | string | optional Teams deployment group |

## Data files — notebooks (`.livemd`)

A notebook is a plain Markdown file (`.livemd`) with two kinds of cells:
**Markdown** cells (free Markdown) and **Code** cells (` ```elixir ` fenced
blocks). Sections are `##` headings; each cell may carry metadata comments.
Notebooks are parsed/rendered by `lib/livebook/notebook/` and
`lib/livebook/live_markdown/`. This is the interchange format for
open/save/export and for hubs.

## Config schema — `LIVEBOOK_*` environment variables

Parsed centrally in `Livebook.config_runtime/0` (`lib/livebook.ex`).

| Variable | Purpose |
|----------|---------|
| `LIVEBOOK_SECRET_KEY_BASE` | Signing/encryption base for cookies |
| `LIVEBOOK_COOKIE` | Erlang distribution cookie |
| `LIVEBOOK_IP` / `LIVEBOOK_PORT` | Bind address / HTTP port |
| `LIVEBOOK_BASE_URL_PATH` / `LIVEBOOK_PUBLIC_BASE_URL_PATH` | Reverse-proxy path prefixes |
| `LIVEBOOK_PASSWORD` | Admin password (≥12 chars) |
| `LIVEBOOK_TOKEN_ENABLED` | Enable/disable token admin auth |
| `LIVEBOOK_IDENTITY_PROVIDER` | Instance ZTA provider (`basic_auth:…`, `cloudflare:…`, `google_iap:…`, `tailscale:…`, `custom:Module`) |
| `LIVEBOOK_IFRAME_PORT` / `LIVEBOOK_IFRAME_URL` | Iframe app endpoint |
| `LIVEBOOK_TEAMS_URL` / `LIVEBOOK_TEAMS_AUTH` | Livebook Teams sync |
| `LIVEBOOK_HOME` / `LIVEBOOK_DATA_PATH` / `LIVEBOOK_APPS_PATH` | Storage locations |
| `LIVEBOOK_DEFAULT_RUNTIME` | Default runtime spec |
| `LIVEBOOK_CLUSTER` | `DNSCluster` query for clustering |
| `LIVEBOOK_NODE` / `LIVEBOOK_AGENT_NAME` | Distribution node / deploy agent |
| `LIVEBOOK_FORCE_SSL_HOST` / `LIVEBOOK_PROXY_HEADERS` | TLS redirect / proxy header trust |
| `LIVEBOOK_ALLOW_URI_SCHEMES` | Permitted URI schemes |
| `LIVEBOOK_AWS_CREDENTIALS` | Enable AWS credential chain |
| `LIVEBOOK_APP_SERVICE_NAME` / `LIVEBOOK_APP_SERVICE_URL` | App service integration |
| `LIVEBOOK_IMAGE_REGISTRY_URL` | Custom app image registry |

## Interface schemas

### HTTP / LiveView routes (`lib/livebook_web/router.ex`)

| Scope | Auth pipeline | Examples |
|-------|---------------|----------|
| `/public` | none | `GET /health`, session asset/audio/image inputs |
| `/dev` | `:api` | `POST /dev/{sync,open,restamp}` (JSON; reject `Origin`) |
| `/` (default live_session) | `:browser,:auth` | `/`, `/open/:tab`, `/settings`, `/hub/:id`, `/sessions/:id`, `/learn`, `/apps-dashboard` |
| `/apps/:slug` (apps live_session) | `:browser,:user` | app pages + app sessions |
| `/authenticate`, `/logout` | `:user` | auth controller |

### Sockets / channels

- `/live` — `Phoenix.LiveView.Socket` (WebSocket, `check_origin: false`, CSRF-guarded).
- `/socket` — `LivebookWeb.Socket`; `js_view_channel` bridges JS widgets
  (only user-initiated events are proxied).

### Dev endpoints (`docs/dev_endpoints.md`)

JSON request/response: `{"status":"ok"}` or
`{"status":"error","message":"…"}`.

### Protobuf (Livebook Teams sync)

`proto/messages.proto` + `proto/lib/livebook_proto/*.pb.ex` define
`Proto.Message` payloads — org/deployment-group/secret/filesystem/app-deployment
events, agents, billing status, and errors — exchanged with the Teams hub
server. Top-level envelope is `Proto.Message`.

## Summary

No relational schema; durable state is an ETS dump (`livebook_config.v*.ets`)
managed by `Livebook.Storage` across named keyspaces, plus filesystem data
(notebooks, secret-encrypted hub config, dependency caches), `LIVEBOOK_*` env
config, and the HTTP/LiveView/socket/protobuf interface schemas above.
