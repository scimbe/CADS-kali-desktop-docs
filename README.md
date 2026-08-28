# CADS Kali Desktop — documentation

Diátaxis-structured MkDocs Material site for
[scimbe/CADS-kali-desktop](https://github.com/scimbe/CADS-kali-desktop) — a gated, browser-reachable
Kali Linux desktop behind a CADS-Tunnel agent.

**Live:** https://scimbe.github.io/CADS-kali-desktop-docs/

Every claim is marked **measured** (run and observed), **documented in repo** (stated by the
source files/README but not independently re-run while writing these docs), or **not verified
here**. Nothing is presented as confirmed-working that wasn't actually observed.

## Local preview

```bash
pip install mkdocs-material
mkdocs serve
```

## Layout

- `docs/explanation/` — the gate and access-control model: the four containers, why each exists
- `docs/how-to/` — deploy and redeploy, including the single-file bind mount gotcha
- `docs/reference/` — every environment variable, the container/port layout, both request paths

Built and deployed by `.github/workflows/docs.yml` on every push to `main`.
