# Project Architecture — Summary

Fork of upstream Livebook (`livebook-dev/livebook`): an Elixir/Phoenix app
serving interactive code notebooks. At the checked-out SHA (`0.20.0-dev`) there
is no in-repo source divergence from upstream.

- **Shape**: browser ⇄ Phoenix endpoint/router ⇄ LiveView pages ⇄ per-notebook
  `Livebook.Session` ⇄ a separate Erlang **runtime** node (standalone, attached,
  Fly, K8s, embedded). Runtime HTTP is proxied via `Proxy.Server`/`Handler`.
- **Boot**: `Livebook.Application` starts a `:one_for_one` tree — PubSub, task
  supervisor, migration step, Tracker, runtime/session DynamicSupervisors, hubs
  registry/supervisor + Teams boot step, `Apps.DeploymentSupervisor`, endpoint
  with the configured ZTA identity provider.
- **Evaluation**: notebooks run in trusted-peer runtime nodes over Erlang
  distribution; `Mix.install/2` supported per notebook.
- **Web/auth**: pipelines `:browser`/`:api`/`:auth`/`:user`; `/public/*`
  unauthenticated. Three auth layers — instance identity provider
  (`LIVEBOOK_IDENTITY_PROVIDER`: basic_auth/cloudflare/google_iap/tailscale/
  custom/Teams), admin token or password, per-app passwords. Cookie `lb_session`
  signed; `authorized?/1` needs `access_type == :full`.
- **Persistence**: no relational DB — filesystem state under LIVEBOOK_HOME /
  LIVEBOOK_DATA_PATH with `Livebook.Storage` ETS-dump EAV cache.
- **Stack**: Elixir 1.20.1 / OTP 28.1.1, Phoenix 1.8, LiveView 1.1, Bandit,
  nimble_zta, protobuf, kubereq, yaml_elixir; bun+vite frontend; Tauri desktop.
- **Deployment (Noizu)**: Terraform `platform/ai/livebook.tf`, single-replica
  `Recreate` Deployment at nb.noizu.com running the **upstream image
  `ghcr.io/livebook-dev/livebook:0.19.10`** (behind this tree), 10Gi RWO PVC,
  Infisical `/ai/livebook` secrets, gzip-asset startup wrapper, Cloudflare-mTLS
  ingress, `/public/health` probes.
