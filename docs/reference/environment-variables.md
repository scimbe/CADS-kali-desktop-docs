# Environment variables

<span class="prov d">documented in repo</span> Sourced directly from `.env.template`,
`.env.tunnel.template`, and the compose files. Secrets are never committed to the repository.

## `.env` — desktop credentials {#env}

From `.env.template`. Used inside the `kali` container (selkies basic auth) and, after
`./setup.sh gate-secret`, injected upstream by the gate so a visitor logs in once.

| Variable | Purpose |
|---|---|
| `KASM_USER` | The desktop's basic-auth username. Also the identity injected server-side by `kali-gate` for SSO. |
| `KASM_PASSWORD` | The desktop's basic-auth password. Should be a random value, deliberately distinct from the Keycloak/gate password — see [why](../explanation/the-gate-and-access-model.md#why-the-desktops-own-password-is-a-separate-random-credential). Never typed by the visitor; injected server-side. |

## `.env.gate` — derived, not hand-written

Written by `./setup.sh gate-secret` from `.env`. Never committed; its value is never printed.

| Variable | Purpose |
|---|---|
| `KASM_BASIC_B64` | `base64(KASM_USER:KASM_PASSWORD)`. Injected by `kali-gate` as the `Authorization: Basic` header on every proxied request, so the desktop's own login prompt never reaches the browser. |

## `.env.tunnel` — tunnel agent configuration {#env-tunnel}

From `.env.tunnel.template`. Filled from the CADS-Tunnel portal's **Install** page after creating
the tunnel.

| Variable | Purpose |
|---|---|
| `CT_AGENT_TOKEN` | Routing token pasted from the portal's Install page. **Secret.** |
| `CT_AGENT_EDGE` | Host:port of the CADS-Tunnel edge this agent dials out to. |
| `CT_AGENT_EDGE_CERT_URL` | URL used to validate the edge's TLS certificate. |
| `CT_AGENT_CP_URL` | Control-plane URL the agent talks to. |
| `CT_AGENT_HOSTNAME` | The public `*.bunsenbrenner.org` hostname this desktop is reachable at. |
| `CT_AGENT_MODE` | `browser` — browser-plane mode (dials out; no inbound port). |
| `CT_AGENT_ORIGIN` | Where the agent forwards traffic once it decrypts it — `kali-desktop:3000` (fallback path) or `kali-gate:3000` (full stack; overridden by `compose.gate.yml`, see [Applying or reverting the gate/revoke layer](../how-to/deploy-and-redeploy.md#applying-or-reverting-the-gaterevoke-layer)). |
| `CT_AGENT_ORIGIN_PROTO` | `tcp` — plain HTTP to the origin; the edge terminates TLS. |
| `CT_AGENT_ID` | Agent identity pasted from the portal's Install page. |
| `CT_AGENT_STATE_DIR` | Where the agent persists its bound identity, so a restart doesn't re-redeem the one-time join token. |
| `CT_AGENT_CAPABILITY_OUT` | Path the agent writes its capability token to. |
| `CT_AGENT_RECONNECT_MAX_ATTEMPTS` | `0` = retry indefinitely. |

## Set directly in `compose.gate.yml` (not secret)

| Variable | Purpose |
|---|---|
| `CT_GATE_UPSTREAM_HOST` | Upstream host the gate calls for `/gate/check` and `/gate/logout` — `bunsenbrenner.org`. |
| `KALI_ALLOWED_EMAIL` | The single identity allowed through, compared against the edge-verified `X-Gate-Email`. Not treated as a secret — it's the operator's own login — so it's kept in the compose file rather than an env file. |
| `KALI_REVOKE_HOST` (on `kali-gate`) | Where `kali-gate`'s second `forward_auth` asks — `http://kali-revoke:8099` (scheme included; Caddy needs it for an upstream). |

## Set directly in `compose.revoke.yml` (not secret)

| Variable | Purpose |
|---|---|
| `KALI_REVOKE_HOST` (on `kali`) | Consumed by the `svc-de/finish` hook, which `POST`s here on every desktop-session teardown — `kali-revoke:8099` (**no scheme** here; it's a bare curl target, not a Caddy upstream — do not confuse with the `kali-gate` variable of the same name above). |
