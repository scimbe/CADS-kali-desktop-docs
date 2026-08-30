# The gate and access-control model

The desktop image (`linuxserver/kali-linux`, selkies/KasmVNC) already streams a KDE session to a
browser with its own basic auth and a persistent `/config` volume. Everything documented on this
page exists to solve three problems that image cannot solve by itself: reachability without an
open port, a gate that's actually enforced, and a logout that actually logs out.

## The four containers

| Container | Compose file | Role |
|---|---|---|
| `kali` (`kali-desktop`) | `compose.kali.yml` | The desktop itself. Persistent `/config`, its own `CUSTOM_USER`/`PASSWORD` basic auth, **no published host port**. |
| `kali-agent` | `compose.tunnel.yml` | A `ct-agent` in browser-plane mode. Dials **out** to the CADS-Tunnel edge and joins the desktop's Docker network to reach it — so nothing needs an inbound firewall rule. |
| `kali-gate` | `compose.gate.yml` | A Caddy instance the tunnel agent forwards to instead of the desktop directly. Calls the edge's `/gate/check`, injects the desktop's basic credential server-side (SSO), and independently compares the verified identity against `KALI_ALLOWED_EMAIL`. |
| `kali-revoke` | `compose.revoke.yml` | A stdlib-only Python sidecar. Holds a one-shot "the desktop session ended" flag that `kali-gate`'s second `forward_auth` turns into a real cookie-clearing logout. |

## Why the portal's "require login" toggle isn't enough on its own

The tunnel portal has a `require_login` switch, but per this repository's own comments, that
switch **does not block browser-plane traffic by itself** — the gate is origin-driven: something
at the origin has to call `/gate/check` and act on the result. `Caddyfile.gate` is that something.
Without a proxy in front of the desktop, the access list configured in the portal is inert, and
the desktop's own shared password would be the *only* protection. `kali-gate` is what makes the
tunnel's access list actually enforced for this origin — the same pattern used in front of the
`sort` demo's origin.

## Why the gate re-checks identity instead of trusting a bare 200

`Caddyfile.gate` calls `/gate/check` at the edge, but it does not treat "the edge said 200" as
sufficient permission on its own. It additionally compares the edge-verified `X-Gate-Email`
header against `KALI_ALLOWED_EMAIL` and refuses on mismatch or absence. Per the README, this was
deliberate defense-in-depth: a real second account (signed in, on a *different* tunnel's access
list but not this one's) was refused by the edge itself before ever reaching this origin — which
refuted the concern that a bare `200` might admit any signed-in account. The identity check stays
anyway, because that upstream enforcement lives in someone else's config (the portal), one toggle
away from being switched off; comparing the identity here makes admission depend on *this*
repository's config too, not just the portal's.

The check fails closed by construction: the client-supplied copy of `X-Gate-Email` is stripped
(`request_header -X-Gate-Email`) *before* `forward_auth` runs, inside a `route {}` block — Caddy
executes directives in priority order rather than textual order, so without the `route {}` the
strip could run after the real value was set, deleting a legitimate identity instead of a spoofed
one. A missing header doesn't match the allow-list matcher, and `not` turns that into a refusal.

## Why the desktop's own password is a separate, random credential

`KASM_PASSWORD` is documented as deliberately **not** a copy of the Keycloak/gate identity's
password. The visitor never types it — the gate injects it server-side via
`header_up Authorization "Basic {$KASM_BASIC_B64}"` — so there's nothing to gain from it being
memorable, and reusing the Keycloak password would collapse two independent layers into one: a
single leaked credential would then open both.

## Why a separate revoke sidecar exists at all

Logging out *inside* the Kali desktop only ends the X/Wayland session. The browser still holds a
valid, `HttpOnly`, edge-issued gate cookie scoped to `.bunsenbrenner.org` — nothing inside the
container can invalidate that cookie directly. What the container *can* do is report that the
session ended:

1. The desktop's s6 `svc-de/finish` hook (which fires on **every** teardown route — the Leave
   menu, `ksmserver`, `loginctl`, or a crash, not just one menu item) `POST`s to
   `kali-revoke:8099/revoke`.
2. `kali-revoke` holds that fact as an in-memory, one-shot flag.
3. `kali-gate`'s second `forward_auth` asks `kali-revoke`'s `/check` on every request. A `401`
   response makes Caddy redirect the browser to `/logout`, which clears the actual gate cookie.

Two details in `revoke.py` are there because a naive version of this genuinely failed in
practice (per its own comments, reproduced here since they explain real, previously-hit bugs):

- **The flag is only consumed by a document-level page load**, not any request. Because the
  desktop page is a live stream, the *first* request to reach `/check` after a logout is often a
  background subresource fetch, not the page navigation — a naive one-shot flag got burned there,
  redirected invisibly, and the stream's own reconnect a moment later saw a clean `200`. The user
  was left looking at a fully working desktop after logging out. `revoke.py` distinguishes the two
  using the browser-sent `Sec-Fetch-Dest: document` header, refusing (but not consuming) on
  anything else.
- **The desktop page carries its own "am I still welcome?" probe** (`hooks/selkies-index.html`)
  that polls `/gate-ping` and reloads on a non-200, so a revoked tab visibly leaves the desktop
  within seconds instead of continuing to paint the last frame it already received.

## Which mode is actually live

The repository documents **two** distinct deployment shapes, and they are not the same:

<div class="measured" markdown>
<span class="prov m">measured</span> **The full, Keycloak-gated stack is what's live.** `compose.gate.yml`'s
own header comment reads *"Keycloak gate in front of the Kali desktop — **PREPARED, NOT
DEPLOYED** (awaiting operator go-ahead)"* — that line is a stale comment nobody updated after the
gate actually shipped, not a description of current state. Two independent, live checks confirm
the gated stack is what's actually running:

- `docker inspect` on the running `kali-gate` container shows its Compose project's config-files
  list includes `compose.gate.yml` and `compose.revoke.yml` alongside `compose.tunnel.yml` — the
  gate and revoke layers are part of the deployed stack, not merely present in the repo.
- `curl -sI https://kali.bunsenbrenner.org/` returns a real `302` with `Via: 1.1 Caddy` and
  `Cache-Control: no-store`, redirecting to
  `bunsenbrenner.org/gate/start?host=kali.bunsenbrenner.org&return=%2F` — the actual Keycloak
  login redirect, not the fallback path's bare `401 WWW-Authenticate: Basic`. Following that
  redirect once more lands on `auth.bunsenbrenner.org/realms/ct-demo/protocol/openid-connect/auth`
  (`client_id=ct-portal`), i.e. the real Keycloak realm. This also matches the standing
  stability-watch's health-check baseline, which has consistently observed `kali` returning a
  302-unauthenticated response. Re-confirmed live 2026-08-29.
</div>

So: the top-level `README.md`'s *"Built and verified live on 2026-08-17"* claim is the accurate
one for the gate/revoke layer. `compose.gate.yml`'s "PREPARED, NOT DEPLOYED" header should be
treated as outdated documentation inside that file, not as a signal about what's running.

Architecturally, the two modes compose additively, and the deployed config files confirm the
**full stack** is the one in use:

- **Fallback (documented, not currently deployed):** `compose.kali.yml` + `compose.tunnel.yml` —
  desktop + tunnel agent only, gated solely by the edge's `require_login` and the desktop's basic
  auth.
- **Full stack (confirmed live):** the above **plus** `compose.gate.yml` + `compose.revoke.yml` —
  adds the Keycloak gate, SSO credential injection, identity re-check, and real logout semantics.

See [Deploy and redeploy](../how-to/deploy-and-redeploy.md) for the commands that apply or revert
each layer.
