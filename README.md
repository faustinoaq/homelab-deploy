# hl: git-push deploys to your homelab

A tiny, self-hosted alternative to Vercel/Netlify for people who already have a few
Docker VMs at home and a Cloudflare tunnel. One Python file, stdlib only.

```
hl deploy ~/code/myapp --route myapp=3100 --watch
# → https://myapp.example.com, redeployed on every `git push`
```

Built to be driven by LLM agents as well as humans: every command has `--json` and
`--dry-run`, nothing prompts, exit codes mean something, secrets are never echoed.

## How it works

```
laptop ── git push ──▶ GitHub (private repo = off-site backup of the source)
   │                      ▲
   │ ssh (via jump host)  │ git fetch with a per-app read-only deploy key
   ▼                      │
docker VM: ~/apps/<name>  ──▶ docker compose up -d --build
                                │
Cloudflare tunnel (remotely managed): <sub>.<zone> ─▶ http://<vm-ip>:<port>
```

`hl deploy <dir>`:

1. **Preflight**: `cf` and `gh` installed and logged in; the dir is a git repo root with
   a GitHub `origin` (or `--create-repo` makes a private one); no uncommitted changes.
2. **Push** the current branch to GitHub.
3. **Deploy key**: creates `~/.ssh/hl_<name>` on the VM and registers it as a
   *read-only* deploy key on the repo (`gh repo deploy-key add`). Idempotent.
4. **Checkout**: clones into `~/apps/<name>` and installs `.git/hl/{pull,up}.sh` plus a
   `post-merge` hook (so a manual `git pull` on the server redeploys too).
5. **Secrets**: untracked `.env*` files are copied over SSH (mode 600), never committed.
6. **Build & run**: `docker compose -p <name> up -d --build --remove-orphans`
   (or `docker build` + `docker run` for Dockerfile-only projects).
7. **Watch** (optional, `--watch`): a cron job on the VM runs `pull.sh` every 2 minutes;
   when `origin/<branch>` moved it resets and rebuilds, unless every changed file is outside the
   app's subdirectory or matches `ignore`. No inbound webhook needed. Changing the manifest needs
   one `hl deploy` to regenerate the scripts on the VM.
8. **Publish**: adds a tunnel ingress rule (before the catch-all) and a proxied CNAME.
9. **Smoke test**: polls the HTTPS URL; exit code 2 if it never answers.

## Requirements

**Workstation:** Python 3.8+, `git`, `ssh`, `curl`, [`gh`](https://cli.github.com)
(logged in), the Cloudflare CLI [`cf`](https://github.com/cloudflare/cf) (logged in).

**Each VM:** an unprivileged deploy user with your SSH key in `~/.ssh/authorized_keys`,
member of the `docker` group, plus `git`, Docker with `docker compose` v2, and `crontab`
(only for `--watch`). No root login and no sudo needed.

**Cloudflare:** a *remotely-managed* tunnel (config source "cloudflare", i.e. created in the
dashboard / Zero Trust) whose connector can reach the VMs.

## Setup

```bash
git clone https://github.com/faustinoaq/homelab-deploy && cd homelab-deploy
ln -s "$PWD/hl" ~/.local/bin/hl
mkdir -p ~/.config/hl && cp config.example.json ~/.config/hl/config.json   # edit it
cf tunnels list            # tunnel_id
cf zones list --name example.com   # zone_id
hl ssh-config >> ~/.ssh/config     # optional: `ssh docker1` just works
hl doctor
```

Config is looked up in `$HL_CONFIG`, `~/.config/hl/config.json`, then `./config.json`.
`jump` is optional (omit it if the VMs are directly reachable).

## Commands

| command | what it does |
|---|---|
| `hl doctor` | checks cf, gh, git, tunnel, jump host, and docker/git/cron on every VM |
| `hl routes` | public hostname → VM:port table |
| `hl ps [--host H]` | containers on one/all VMs |
| `hl ports --host H` | listening ports (pick a free one) |
| `hl deploy [dir] [--route SUB=PORT]… [--watch [MIN]] [--create-repo]` | see above |
| `hl logs NAME [--service S] [--deploy]` | container logs, or the auto-deploy log |
| `hl unwatch NAME` | stop auto-redeploy |
| `hl down NAME [--volumes]` | stop the app (routes kept); `--volumes` deletes data |
| `hl expose SUB PORT --host H` / `hl unexpose SUB` | manage routes only |
| `hl exec --host H -- CMD` | run something on a VM |

Global flags: `--json`, `--dry-run`.

## Per-project manifest (`homelab.json`, optional)

```json
{
  "name": "myapp",
  "host": "docker3",
  "branch": "main",
  "compose": "docker-compose.prod.yml",
  "services": ["db", "backend", "frontend"],
  "routes": { "myapp": 3000, "myapp-api": 8000 },
  "watch": true
}
```

Other keys:

* `"port"` / `"container_port"`: Dockerfile-only projects (host port / port inside the container).
* `"env_files"`: which untracked env files to copy (default: every `.env*` except `*.example`).
* `"health_path"`: path the smoke test requests (default `/`; any status except 404/5xx counts as up).
* `"ignore"`: glob patterns (relative to the app directory) whose changes don't trigger an
  auto-redeploy, e.g. `["docs/**", "*.md", "android/**"]`. A push redeploys only if at least one
  changed file matches none of them. In these patterns `*` also matches `/`. `hl deploy` always deploys.

CLI flags override the manifest. See `examples/`.

### Monorepos

Run `hl deploy path/to/app` on a subdirectory with its own `homelab.json`: the whole repo is
cloned, the build runs in that subdirectory, and `--watch` only rebuilds when a push touches it.

## Using it from an AI agent

`skill/homelab-deploy/SKILL.md` is a [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills):

```bash
ln -s "$PWD/skill/homelab-deploy" ~/.claude/skills/homelab-deploy
```

Other agents: point them at `hl --help` and tell them to always use `--json`.

## Safety

* The tunnel config is read-modify-written; the previous version is saved to
  `~/.local/state/hl/tunnel-config-v<N>-<ts>.json` first.
* Existing hostnames are never repointed, and foreign DNS records never replaced, without `--force`.
* Deploy keys are read-only and per app; the VM never gets your GitHub credentials.
* An existing non-git `~/apps/<name>` is moved aside (`.pre-hl.<ts>`), never deleted.
* Only `down --volumes` deletes data.

## License

MIT
