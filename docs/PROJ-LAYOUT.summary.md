# Project Layout — Summary

Fork of upstream Livebook (`livebook-dev/livebook`); Elixir/Phoenix notebook
web app. No Noizu source divergence at this SHA.

```
livebook/
├── lib/                  # App source: livebook.ex + livebook/, livebook_web/, livebook_cli/, mix/tasks/
├── config/               # config.exs, dev/test/prod.exs, runtime.exs (LIVEBOOK_* env)
├── rel/                  # OTP release templates (server/) + Tauri desktop app (app/)
├── proto/                # Protobuf messages for Livebook Teams sync
├── iframe/               # Separate Mix project (iframe port)
├── assets/               # Frontend: bun + vite (css/, js/, public/, test/)
├── docs/                 # Upstream docs + PROJ-* / THREAT-MODEL
├── test/                 # ExUnit suite
├── priv/                 # Static/generated runtime files
├── .github/workflows/    # ci.yml, release.yml
├── Dockerfile            # Multi-stage build (default + CUDA), release image
├── mix.exs / mix.lock    # Project + locked deps
├── versions              # Pinned elixir/otp/rebar3/ubuntu
├── README.md / RELEASE.md / SECURITY.md / CHANGELOG.md / LICENSE
├── .editorconfig / .formatter.exs / .dockerignore / .gitattributes
```

lib/ core: `application.ex` (OTP boot), `storage.ex` (ETS-dump EAV store),
`apps/` (deployments), `runtime/` (evaluators), `session/`, `notebook/`,
`hubs/`, `secrets/`, `teams/`, `zta/` (Teams ZTA), `k8s/`, `proxy/`,
`file_system/`; web layer in `livebook_web/` (router, endpoint, live/,
plugs/, channels/).
