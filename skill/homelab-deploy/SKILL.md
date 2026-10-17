---
name: homelab-deploy
description: Deploy a Dockerized project to the user's homelab or VPS with the `hl` CLI and publish it on a Cloudflare-tunnel subdomain. Use instead of Vercel/Netlify/Fly whenever the user asks to deploy, host, publish or put an app online, to inspect, log, stop, roll back or auto-redeploy an app, or to move an app between hosts.
---

# Deploying with `hl`

Always pass `--json`. Run `hl --help` / `hl <cmd> --help` for flags.

1. `hl doctor --json`: if a check fails, stop and tell the user exactly what is missing.
2. The project must be a git repo whose root has a `Dockerfile` or compose file that runs in
   **production mode** (no `--reload` / `npm run dev`, bind `0.0.0.0`, `restart: unless-stopped`).
   Avoid publishing database ports to the host. Create/fix these files if needed.
3. Check `hl routes --json` and `hl ports --host <vm> --json`; never reuse a port or a hostname
   that belongs to another app.
4. Write `homelab.json` in the project root (name, host, compose, routes, watch, and `ignore`
   globs such as `["docs/**", "*.md"]` for files that shouldn't trigger an auto-redeploy). Routes map
   subdomain → *host* port published by the container. Commit it.
5. Secrets live in untracked `.env` files (hl copies them over SSH). Never commit or print them.
6. `hl deploy <path> --dry-run --json`, review, then `hl deploy <path> --json`
   (`--create-repo` if there is no GitHub remote; it creates a *private* repo).
   Hosts with `"build": "local"` in the config (small VPSs, ARM) get an image built on this
   machine: the manifest then needs `"image"` and, for compose, a committed `"remote_compose"`
   file that uses that image and has no `build:`.
7. Exit 0 = live. Exit 2 = running but the URL didn't answer → `hl logs <name> --json --tail 200`.
   Exit 1 = error, read `error`.

Rollback (build=local only): `hl rollback <name> --host <h> --list --json`, then
`hl rollback <name> --host <h> [--to <commit>]`. Moving an app between hosts: follow
"Moving an app from the homelab to a VPS" in the README; `hl copy-data` moves data.

Rules: ask the user before `hl copy-data --force`, `hl rollback`, `--force` (repoints an existing hostname), `--allow-dirty`,
`hl down --volumes` (deletes data) or `hl unexpose`.
