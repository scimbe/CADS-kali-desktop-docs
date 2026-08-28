# Deploy and redeploy

This follows `setup.sh` and the compose files in
[`scimbe/CADS-kali-desktop`](https://github.com/scimbe/CADS-kali-desktop) directly. Nothing here
is an invented step — every command below is one `setup.sh` actually runs, or one its own README
documents.

<span class="prov d">documented in repo</span> These are the repository's documented steps; this
docs pass did not execute them against a live host.

## Prerequisites

- Docker with Compose v2 (`docker compose version`).
- A tunnel created in the CADS-Tunnel portal, with `require_login=1` and your account on its
  access list, then **Install** clicked to obtain the routing tokens.

## 1. Desktop credentials

```bash
git clone https://github.com/scimbe/CADS-kali-desktop.git && cd CADS-kali-desktop
cp .env.template .env && $EDITOR .env      # set KASM_USER, KASM_PASSWORD
chmod 600 .env
```

`KASM_PASSWORD` should be a random value, deliberately **not** your Keycloak/gate password — see
[why](../explanation/the-gate-and-access-model.md#why-the-desktops-own-password-is-a-separate-random-credential).

## 2. Derive the gate's injected credential

```bash
./setup.sh gate-secret
```

Reads `.env`, writes `.env.gate` (base64 of `user:password`, `chmod 600`, value never printed).
This is what lets the gate log the visitor in once and never show the desktop's own login prompt.

## 3. Tunnel configuration

```bash
cp .env.tunnel.template .env.tunnel && $EDITOR .env.tunnel   # paste the portal's Install tokens
chmod 600 .env.tunnel
```

See [Environment variables](../reference/environment-variables.md#env-tunnel) for what each field
means. Note `CT_AGENT_ORIGIN`: whatever you put here is silently overridden to `kali-gate:3000`
by `compose.gate.yml` when you bring up the full stack (step 5 below) — see
[Applying or reverting the gate/revoke layer](#applying-or-reverting-the-gaterevoke-layer). Fine
to leave at its template default; just don't be surprised it isn't the value actually in effect.

## 4. Set who is allowed in

In `compose.gate.yml`, set `KALI_ALLOWED_EMAIL` to the **same address** you put on the tunnel's
access list in the portal. This is not treated as a secret in this repo (it's your own login
identity) — it stays in the compose file, in plain sight, rather than in an env file.

<span class="prov m">measured</span> **This is the one step `setup.sh check` cannot catch for
you if you skip it.** The tracked `compose.gate.yml` ships with a real, non-placeholder address
already filled in (the upstream repo owner's own) — `check`'s validation is `[ -n
"$KALI_ALLOWED_EMAIL" ]`, non-empty only, confirmed by reading `setup.sh` directly. A value that
is merely non-empty always passes, whether or not it's *your* address. If you deploy a fork of
this repo without editing `KALI_ALLOWED_EMAIL` here, `check` and `verify` both report
green while the gate silently keeps admitting the original owner's identity — and yours, on the
list you configured in the portal, never gets in. There is no automated guard against this;
double-check this line by eye before `up`.

## 5. Check, then bring it up

```bash
./setup.sh check     # validates files, secrets, permissions, hostname, compose config, Caddyfile — changes nothing
./setup.sh up         # docker compose up -d --build ; first run pulls ~12 GB
```

`setup.sh check` refuses to proceed if `.env`, `.env.tunnel`, or `.env.gate` are missing or not
`chmod 600`.

By default `setup.sh` brings up the **full stack** — `compose.kali.yml` + `compose.tunnel.yml` +
`compose.gate.yml` + `compose.revoke.yml` together (see
[which mode is actually live](../explanation/the-gate-and-access-model.md#which-mode-is-actually-live)
for the distinction between this and the fallback path).

## 6. Verify

```bash
./setup.sh verify
```

Re-measures the security properties documented in the README against the live deployment —
unauthenticated redirect, deep-path coverage, basic-auth-alone insufficiency, the logout target,
the `no-store` header, the revocation one-shot semantics, absence of published ports, presence of
the logout probe in the served page, and that the running finish hook and Caddyfile are the
patched ones — rather than assuming any of them.

## Redeploying after a config change

!!! danger "Single-file bind mounts pin an inode"
    `Caddyfile.gate` and `hooks/svc-de-finish` are bind-mounted as individual files. Editing one
    and then **restarting** the container changes nothing — Docker keeps serving the old,
    already-unlinked file from the pinned inode. This is called out in the repo as having cost
    real debugging time twice in one day. Always **recreate**, not restart:

    ```bash
    docker compose -f compose.kali.yml -f compose.tunnel.yml -f compose.gate.yml -f compose.revoke.yml \
      up -d --force-recreate kali-gate
    # or, for the finish hook:
    docker compose -f compose.kali.yml -f compose.tunnel.yml -f compose.gate.yml -f compose.revoke.yml \
      up -d --force-recreate kali
    ```

    `setup.sh verify` includes a check that the file actually present *inside* the running
    container is the patched one, specifically because of this failure mode.

## Applying or reverting the gate/revoke layer

```bash
# Full stack (gate + revoke on top of desktop + tunnel):
docker compose -f compose.kali.yml -f compose.tunnel.yml -f compose.gate.yml -f compose.revoke.yml up -d

# Revert to the fallback path (desktop + tunnel only, edge-only gating):
docker compose -f compose.kali.yml -f compose.tunnel.yml up -d
```

## Stopping / destroying

```bash
./setup.sh down      # stops everything; the desktop's persistent volume is kept
./setup.sh destroy   # stops AND deletes the persistent desktop volume — irreversible, asks for confirmation
```
