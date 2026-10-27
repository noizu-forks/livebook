# Threat Model — Summary

Livebook accepts and **executes user code** — crown jewels: runtime code
execution, notebook/app secrets and env vars, the admin interface, and the
persisted `/data` volume. Model is for the deployed instance at `nb.noizu.com`
(Cloudflare mTLS → ingress-nginx → single-replica upstream-image pod on a 10Gi
PVC), grounded in `router.ex`, `endpoint.ex`, `plugs/`, `zta/`, `storage.ex`,
upstream auth/dev-endpoint docs, and `livebook.tf`.

**Trust boundaries**: internet→CF(TLS)→ingress(client-cert verify)→pod;
browser→server (WS `check_origin:false` + CSRF); server→runtime (Erlang dist,
trusted peers); server→Teams/hex.pm/AWS; Infisical→pod secret; unauthenticated
`/public/*`; `/dev/*` (disabled).

**Register (12)**: T-001 arbitrary code exec (by design, isolated not sandboxed);
T-002 `/dev/*` no auth beyond Origin (disabled by default); T-003 WS CSWSH
(partial: CSRF); T-004 admin token/password (mitigated via Infisical secret);
T-005 root + writable PVC (accepted); T-006 CF origin client-cert gate
(mitigated); T-007 hub secrets key in same store (partial); T-008 upstream
image tag-not-digest supply chain (partial); T-009 Teams callback CSRF
(mitigated: `teams_state` secure_compare); T-010 import path traversal / env
injection (mitigated: recent fixes); T-011 distribution cookie (mitigated:
random default); T-012 upload/WS DoS (partial: 50m body, 3600s timeouts).

**Coverage**: mitigated T-002/04/06/09/10/11 · partial T-003/07/08/12 ·
by-design T-001 · accepted T-005.

**Residual risk**: code execution is the feature (not a hardened sandbox);
root+writable PVC accepted for single-writer use; upstream image trusted by
tag; `/dev/*` must remain disabled if the instance is exposed.
