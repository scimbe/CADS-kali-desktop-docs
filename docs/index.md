# CADS Kali Desktop

A full **Kali Linux desktop**, streamed to the browser, with a **persistent profile** — reachable
**only through a CADS-Tunnel agent** (no host port is ever published) and, in the repository's
documented end-state, gated by **Keycloak** so only an allow-listed account gets in.

<span class="prov d">documented in repo</span> This page reflects
[`scimbe/CADS-kali-desktop`](https://github.com/scimbe/CADS-kali-desktop) as checked out while
writing these docs. The live deployment mode was independently confirmed — see
[Which mode is actually live](explanation/the-gate-and-access-model.md#which-mode-is-actually-live)
— but most other claims below trace back to the repository's own README and comments rather than
a fresh re-run of every check during this pass.

## What it is

The desktop itself is the `linuxserver/kali-linux` image (selkies/KasmVNC), which already does
the hard part: a KDE session streamed to a browser tab, with its own HTTP Basic auth and a
`/config` volume that survives restarts. Everything else in this repository exists to solve three
problems that image cannot solve on its own:

- **Reachability without an open port.** A `ct-agent` in browser-plane mode dials *out* to the
  CADS-Tunnel edge, so the desktop needs no inbound firewall rule and no `ports:` on the container.
- **A real access gate.** The tunnel portal's "require login" toggle does not, by itself, block
  browser-plane traffic — the gate has to be origin-driven. A small Caddy instance
  (`kali-gate`) calls the edge's `/gate/check` and enforces the result itself.
- **A logout that actually ends access.** Logging out *inside* the desktop only ends the X
  session; the browser still holds a valid gate cookie and would be let straight back in. A
  sidecar (`kali-revoke`) turns an in-desktop logout into a real one.

## What it's for

A personal, single-user Kali desktop for authorized security work, usable from any browser
without installing or maintaining Kali locally. The access model is built around **one** allowed
identity (`KALI_ALLOWED_EMAIL`) — this is not designed as a multi-tenant or shared desktop.

## Start here

<div class="grid cards" markdown>

- :material-lightbulb-on: **[Explanation](explanation/index.md)**

    The gate and access-control model: the four containers, how they relate, and why the design
    has this many layers.

- :material-wrench: **[How-to](how-to/index.md)**

    Deploy or redeploy the stack, based on `setup.sh` and the compose files — including the
    gotcha that costs the most debugging time.

- :material-book-open-variant: **[Reference](reference/index.md)**

    Every environment variable and what it's for, plus the container and port layout.

</div>

## What's real here, and what isn't yet

| Claim | Status |
|---|---|
| Desktop reachable only via the tunnel agent, no published host port | <span class="prov d">documented in repo</span> — `compose.kali.yml` and `compose.tunnel.yml` publish no `ports:` |
| Unauthenticated request redirected to Keycloak; deep paths (`/websockify`) too | <span class="prov m">measured</span> — `/websockify`, `/config/`, and an arbitrary deep path all 302 to the gate with `return=` preserving the original path, confirmed live 2026-08-28 |
| Desktop's own password alone is not sufficient to get in | <span class="prov d">documented in repo</span> — same measurement |
| A non-allow-listed but signed-in account is refused | <span class="prov d">documented in repo</span> — README describes a real second-account test |
| In-desktop logout actually ends access (not just the X session) | <span class="prov d">documented in repo</span> — `revoke/revoke.py` + `hooks/svc-de-finish` implement this; README describes the measurement |
| The Keycloak-gated stack (not the edge-only fallback) is the one currently deployed | <span class="prov m">measured</span> — live `302` to the Keycloak gate, and `compose.gate.yml`/`compose.revoke.yml` confirmed in the running deployment's config files; see [Which mode is actually live](explanation/the-gate-and-access-model.md#which-mode-is-actually-live) |

The repository's own `setup.sh verify` re-measures the security properties above on demand
against a live deployment; running it is the way to confirm current state, not this page.
