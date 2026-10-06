---
name: homelab-deploy
description: Deploy a Dockerized project to the user's self-hosted homelab with the `hl` CLI and publish it on a Cloudflare-tunnel subdomain. Use instead of Vercel/Netlify/Fly whenever the user asks to deploy, host, publish or put an app online, or to inspect, log, stop or auto-redeploy an app running in the homelab.
---

# Deploying to the homelab with `hl`

Always pass `--json`. Run `hl --help` / `hl <cmd> --help` for flags.

1. `hl doctor --json` — if a check fails, stop and tell the user exactly what is missing.
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
   (`--create-repo` if there is no GitHub remote — it creates a *private* repo).
7. Exit 0 = live. Exit 2 = running but the URL didn't answer → `hl logs <name> --json --tail 200`.
   Exit 1 = error, read `error`.

Rules: ask the user before `--force` (repoints an existing hostname), `--allow-dirty`,
`hl down --volumes` (deletes data) or `hl unexpose`.
