# Container and port layout

<span class="prov d">documented in repo</span> Sourced directly from the compose files.

## Containers

| Container | Compose file(s) | Image | Ports | Network |
|---|---|---|---|---|
| `kali-desktop` (service `kali`) | `compose.kali.yml`; rebuilt via `hooks/Dockerfile.kali` when `compose.revoke.yml` is applied | `lscr.io/linuxserver/kali-linux:latest` (or `kali-desktop-gated:local` when the revoke layer is built in) | None published. Serves on `:3000` internally. | `kali_net` (external name `kali_desktop_default`) |
| `kali-agent` | `compose.tunnel.yml` | Built from `./agent` (`ct-agent`, pinned version, fetched at build time) | None. Dials **out** to the edge; no listener. | `kali_net` |
| `kali-gate` | `compose.gate.yml` | `caddy:2-alpine` | `expose: 3000` — internal only, no `ports:` mapping | `kali_net` |
| `kali-revoke` | `compose.revoke.yml` | `python:3.12-alpine` running `revoke.py 8099` | `expose: 8099` — internal only | `kali_net` |

No container in this stack publishes a host port in the tunnel-only end state. The **only**
exception is the optional `compose.verify.yml` override:

```yaml
# compose.verify.yml — build-verification only, never part of the real deploy
kali:
  ports:
    - "127.0.0.1:3080:3000"   # KasmVNC HTTP, loopback only
```

Loopback-only, used for driving the desktop with a local browser/Playwright while developing —
explicitly not part of the tunnel-only deployment.

## Request path (full stack: gate + revoke applied) — confirmed live

```
browser
  → CADS-Tunnel edge (TLS termination, /gate/check)
  → kali-agent (browser-plane, dials out; CT_AGENT_ORIGIN=kali-gate:3000)
  → kali-gate (Caddy :3000)
      - strips any client-supplied X-Gate-Email
      - forward_auth edge /gate/check → copies verified X-Gate-Email back
      - refuses if X-Gate-Email != KALI_ALLOWED_EMAIL
      - forward_auth kali-revoke:8099/check → redirects to /logout on 401
      - reverse_proxy → kali-desktop:3000, injecting Authorization: Basic {KASM_BASIC_B64}
  → kali-desktop:3000 (KasmVNC/selkies, :3000 internal)
```

## Request path (fallback: gate/revoke not applied) — documented, not currently deployed

```
browser
  → CADS-Tunnel edge (TLS termination, edge's own require_login=1 check)
  → kali-agent (browser-plane; CT_AGENT_ORIGIN=kali-desktop:3000)
  → kali-desktop:3000 (KasmVNC/selkies own basic auth is the only origin-side check)
```

The full-stack path above is the one confirmed live — see
[Which mode is actually live](../explanation/the-gate-and-access-model.md#which-mode-is-actually-live)
for the evidence. The fallback path is documented for completeness (it's what `compose.tunnel.yml`
alone produces) but is not what's currently deployed.

## Volumes

| Volume | Owner | Contents |
|---|---|---|
| `kali_config` | `kali-desktop` (`compose.kali.yml`) | The persistent desktop profile: home directory, installed tools, settings. Survives `setup.sh down`; only `setup.sh destroy` removes it. |
| `kali_agent_state` | `kali-agent` (`compose.tunnel.yml`) | The agent's bound identity, so a restart doesn't re-redeem the one-time `CT_AGENT_JOIN_TOKEN`. |
| `kali_gate_data` | `kali-gate` (`compose.gate.yml`) | Caddy's own data directory (mounted at `/data`). |
