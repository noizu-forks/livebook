# Project Schema — Summary

Livebook has **no relational database** — no SQL ERD, no Liquibase, no Ecto
migrations. Ecto is used only for embedded schemas/changesets.

- **Persistence**: `Livebook.Storage` — an EAV store in an ETS table persisted
  to `#{data_path}/livebook_config.v<version>.ets`. Keyspaces: `:system`,
  `:settings`, `:env_vars`, `:hubs`, `:default_hub`, `:secrets`,
  `:filesystem`, `:notebook_manager`; personal hub adds `:hub_secrets` and
  `:file_systems` (secrets encrypted with a 64-byte hub key).
- **Filesystem data**: notebooks (`.livemd`), data path (ETS dump + app data),
  apps path, `MIX_INSTALL_DIR` dep cache, temp. Deployed instance maps these
  under `/data/{home,notebooks,app-data,.mix-installs}` (10Gi PVC).
- **Secrets**: `Livebook.Secrets.Secret` embedded schema — `name` (upper-cased
  PK, `^\w+$`), `value`, `hub_id`, `deployment_group_id`.
- **Notebook format**: Markdown `.livemd` with Markdown cells and ` ```elixir `
  code cells, `##` sections.
- **Config**: `LIVEBOOK_*` env vars parsed in `Livebook.config_runtime/0`
  (secret key base, cookie, ip/port, password, token, identity provider,
  iframe, teams, home/data/apps paths, default runtime, cluster, node, ssl,
  proxy headers, uri schemes, aws creds, app service, image registry).
- **Interfaces**: HTTP/LiveView routes (public `/public`, `/dev` JSON,
  authed `/`+`/apps`), sockets `/live` + `/socket` (`js_view_channel`),
  dev endpoints JSON, and protobuf `Proto.Message` set for Livebook Teams.
