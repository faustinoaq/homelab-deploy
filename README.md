# hl: deploy Docker apps to your homelab or a VPS

A small, self-hosted alternative to Vercel or Netlify for people who run their own Docker
machines behind Cloudflare tunnels. One Python file, standard library only.

```
hl deploy ~/code/myapp --route myapp=3100
# https://myapp.example.com, deployed from your laptop over SSH
```

The same config can hold homelab VMs behind a jump host and VPSs reached directly, each with
its own tunnel, so moving an app from one to the other is a manifest change plus one deploy.

Deploys are manual by default: nothing changes on a server unless you run `hl deploy`.
`--watch` is opt-in and adds a cron poll that redeploys on every `git push`. It is convenient,
but whatever lands on GitHub then runs on your server, so weigh that first.

Built to be driven by LLM agents as well as humans: every command has `--json` and
`--dry-run`, nothing prompts, exit codes mean something, and secrets are never printed.

## Build modes

`hl deploy` builds the image in one of two places. Pick per app (`"build"` in the manifest),
per host (`"build"` in the config) or per run (`--build`). The default is `remote`.

| | `remote` | `local` |
|---|---|---|
| Where the image is built | on the server | on your machine |
| What the server gets | a git clone (read-only deploy key) | only the image, the compose file and env files |
| Needs on the server | Docker, compose, git | Docker, compose |
| CPU architecture | the server's own | cross-built for the server (`uname -m`) |
| Auto-deploy (`--watch`) | yes | no (nothing on the server to poll) |
| Rollback | revert and deploy | `hl rollback`, images are kept per commit |
| Good for | homelab VMs with spare CPU and disk | small VPSs, ARM instances, production |

### remote

```
laptop -- git push --> GitHub
   |                     ^
   | ssh                 | git fetch with a per-app read-only deploy key
   v                     |
server: ~/apps/<name> --> docker compose up -d --build
```

1. Creates `~/.ssh/hl_<name>` on the server and registers it as a read-only deploy key.
2. Clones into `~/apps/<name>` and installs `.git/hl/pull.sh`, `up.sh` and a `post-merge` hook.
3. Copies untracked `.env*` files over SSH (mode 600).
4. Runs `docker compose -p <name> up -d --build --remove-orphans`, or `docker build` and
   `docker run` for a project with only a Dockerfile.

### local

```
laptop: git archive <commit> -> docker build --platform <server arch>
   |
   | docker save | gzip | ssh 'gunzip | docker load'
   v
server: ~/<remote_dir> --> docker compose up -d --no-build
```

1. Exports the exact commit being deployed (`git archive`), so uncommitted files never reach
   the image.
2. Asks the server for its architecture and builds with `--platform linux/amd64` or
   `linux/arm64`. The image is tagged twice: the name the server runs (`"image"`, for example
   `myapp:prod`) and `myapp:<commit>`.
3. Streams the image over SSH. The server never runs a build, so a build that runs out of
   memory can't take the live app down with it.
4. Sends the compose file (`"remote_compose"`) and env files, then runs
   `docker compose -p <name> up -d --no-build --remove-orphans`, or `docker run` for a
   Dockerfile-only app.
5. Keeps the images of the last `"keep_images"` releases (default 5) for `hl rollback` and
   deletes older ones.

Cross-building for ARM on an x86 machine works without emulation when the Dockerfile runs its
build steps on the builder's platform (`FROM --platform=$BUILDPLATFORM ...`) and the final
stage only copies files. Otherwise install emulation once:
`docker run --privileged --rm tonistiigi/binfmt --install arm64`. `hl doctor` tells you which
case you are in.

### Every deploy, both modes

1. **Preflight**: `cf` and `gh` logged in, the directory is in a git repo with a GitHub `origin`
   (`--create-repo` makes a private one).
2. **Choose the commit**. On the target branch: refuse uncommitted changes, push HEAD
   (fast-forward only) and deploy it. On any other branch, or with `--no-push`: deploy
   `origin/<branch>` exactly as it is on GitHub and push nothing.
3. **Build and start** as described above.
4. **Health check on the server**: polls `127.0.0.1:<port><health_path>` on the server itself
   for up to a minute. If it never answers, the deploy fails and prints the last 30 log lines.
5. **Publish**: adds a tunnel ingress rule and a proxied CNAME for each route.
6. **Public check**: polls the HTTPS URL. Exit code 2 if it never answers.
7. **Record** the release in `~/.hl/apps/<name>.json` and `<name>.log` on the server.

## Requirements

**Workstation:** Python 3.8+, `git`, `ssh`, `curl`, [`gh`](https://cli.github.com) and the
Cloudflare CLI [`cf`](https://github.com/cloudflare/cf), both logged in. For `build=local`,
Docker with buildx.

**Each server:** a user with your SSH key that can run Docker (member of the `docker` group),
and Docker with `docker compose` v2. With `build=remote` also `git`, and `crontab` for `--watch`.
No root login and no sudo needed.

**Cloudflare:** one or more remotely-managed tunnels (created in the dashboard or with the API,
config source "cloudflare"). A connector can serve several hosts over the network, or run on
the host it serves.

## Setup

```bash
git clone https://github.com/faustinoaq/homelab-deploy && cd homelab-deploy
ln -s "$PWD/hl" ~/.local/bin/hl
mkdir -p ~/.config/hl && cp config.example.json ~/.config/hl/config.json   # edit it
cf tunnels list                    # tunnel ids
cf zones list --name example.com   # zone_id
hl ssh-config >> ~/.ssh/config     # optional: `ssh docker1` then works
hl doctor
```

Config is looked up in `$HL_CONFIG`, `~/.config/hl/config.json`, then `./config.json`.

## Config

```json
{
  "jump": "user@192.168.0.10",
  "apps_dir": "apps",
  "default_host": "docker1",
  "hosts": {
    "docker1": { "ip": "192.168.122.10", "user": "deploy" },
    "vps": {
      "ip": "100.64.0.5",
      "user": "ubuntu",
      "jump": null,
      "build": "local",
      "tunnel_id": "11111111-1111-1111-1111-111111111111",
      "service_host": "localhost"
    }
  },
  "cloudflare": {
    "tunnel_id": "00000000-0000-0000-0000-000000000000",
    "zone": "example.com",
    "zone_id": "0123456789abcdef0123456789abcdef"
  }
}
```

| key | meaning |
|---|---|
| `jump` (top level) | SSH jump host used for every host that doesn't set its own |
| `hosts.<h>.jump` | jump host for this host; `null` connects directly (a VPS, or a Tailscale address) |
| `hosts.<h>.build` | default build mode for apps on this host |
| `hosts.<h>.tunnel_id` | tunnel that publishes this host's apps; default `cloudflare.tunnel_id` |
| `hosts.<h>.service_host` | address the tunnel connector uses to reach the app; default the host's `ip`. Use `localhost` when the connector runs on the host itself |
| `apps_dir` | where apps live on servers, relative to the home directory (default `apps`) |

## Commands

| command | what it does |
|---|---|
| `hl doctor` | checks cf, gh, git, every tunnel and jump host, Docker on every host, and whether this machine can build for each `build=local` host |
| `hl routes` | public hostname, service, host and tunnel, across all tunnels |
| `hl ps [--host H]` | containers on one or all hosts |
| `hl ports --host H` | listening ports, to pick a free one |
| `hl deploy [dir] [--host H] [--build remote\|local] [--route SUB=PORT]... [--no-push] [--watch [MIN]] [--create-repo]` | see above |
| `hl rollback NAME --host H [--to COMMIT] [--list]` | `build=local`: run a previous release again (default: the one before the current) |
| `hl copy-data SRC DST [--force]` | copy a directory or Docker volume between hosts; `SRC`/`DST` are `host:path` or `host:volume:NAME` |
| `hl logs NAME [--service S] [--deploy]` | container logs, or the deploy history |
| `hl unwatch NAME` | stop auto-redeploy |
| `hl down NAME [--volumes]` | stop the app (routes kept); `--volumes` deletes its data |
| `hl expose SUB PORT --host H [--force]` / `hl unexpose SUB` | manage routes only |
| `hl exec --host H -- CMD` | run a command on a host |

Global flags, before or after the command: `--json`, `--dry-run`.

## Per-project manifest (`homelab.json`)

```json
{
  "name": "myapp",
  "host": "vps",
  "branch": "main",
  "build": "local",
  "image": "myapp:prod",
  "remote_dir": "myapp",
  "remote_compose": "deploy/vps/docker-compose.yml",
  "routes": { "myapp": 3300 },
  "health_path": "/api/health"
}
```

| key | modes | meaning |
|---|---|---|
| `name` | both | app and compose project name (default: directory name) |
| `host`, `branch` | both | where and what to deploy (default: `default_host`, current branch) |
| `build` | both | `remote` or `local` |
| `compose` | both | compose file (default: the first of `docker-compose.prod.yml`, `compose.prod.yaml`, `docker-compose.yml`, `compose.yaml`, `compose.yml`) |
| `services` | both | only start these compose services |
| `routes` | both | `{ "subdomain": host_port }`, published through the host's tunnel |
| `health_path` | both | path for the server and public checks (default `/`; any status except 404 and 5xx counts as up) |
| `env_files` | both | untracked env files to copy (default: every `.env*` except `*.example`) |
| `remote_dir` | both | directory on the server, relative to home (default `apps/<name>`) |
| `port`, `container_port` | both | Dockerfile-only apps: host port and port inside the container |
| `cpus`, `memory` | both | Dockerfile-only apps: `docker run` limits, e.g. `"0.25"`, `"128m"` |
| `bind` | local | Dockerfile-only apps: `127.0.0.1` or `0.0.0.0` (default `127.0.0.1` when the tunnel connector runs on the host) |
| `image` | local | image name the server runs (default `<name>:latest`); the compose file must use it |
| `remote_compose` | local | compose file sent to the server, read from the deployed commit; it must use `image:` and no `build:` |
| `dockerfile` | local | Dockerfile path (default `Dockerfile`) |
| `build_args` | local | list of `KEY=VALUE` passed as `--build-arg` |
| `keep_images` | local | releases kept for rollback (default 5) |
| `watch` | remote | poll GitHub from cron and redeploy on push |
| `ignore` | remote | globs whose changes don't trigger an auto-redeploy, e.g. `["docs/**", "*.md"]`; `*` also matches `/` |

CLI flags override the manifest. See `examples/`.

### Monorepos

Run `hl deploy path/to/app` on a subdirectory with its own `homelab.json`. With `remote` the
whole repo is cloned and the build runs in that subdirectory, and `--watch` only rebuilds when
a push touches it. With `local` only that subdirectory is exported and built.

## Moving an app from the homelab to a VPS

1. Add the VPS to the config with its own `jump` (or `null`), `tunnel_id`, `service_host` and
   `"build": "local"`. Install `cloudflared` on it as a connector for that tunnel. Run
   `hl doctor`.
2. In the app's manifest set `"host"` to the VPS, `"build": "local"` and `"image"`. If it uses
   compose, commit a compose file for the VPS (with `image:`, no `build:`) and point
   `"remote_compose"` at it. Bind ports to `127.0.0.1` when the connector runs on the VPS.
3. Try it: `hl deploy . --dry-run`.
4. Stop the old copy so its data stops changing: `hl down <name> --host <old>`.
5. Copy the data: `hl copy-data <old>:~/apps/<name>/data vps:~/<name>-data`, or for a named
   volume `hl copy-data <old>:volume:<project>_data vps:volume:<project>_data`. The copy
   streams through your machine and refuses a non-empty destination unless `--force`.
6. Deploy with `hl deploy . --force`. `--force` lets hl move the hostname from the old tunnel:
   it adds the rule to the new tunnel, repoints the DNS record in place (no gap where the name
   doesn't resolve) and then removes the old rule.
7. Check `hl routes` and the site. If something is wrong, `hl deploy . --host <old> --build remote --force`
   moves it back while the old data is still there.

## Using it from an AI agent

`skill/homelab-deploy/SKILL.md` is a Claude Code skill. Link it into `~/.claude/skills/`. Other
agents: point them at `hl --help` and tell them to use `--json`.

## Safety

* Tunnel configs are read-modify-written; the previous version is saved to
  `~/.local/state/hl/tunnel-config-<tunnel>-v<N>-<ts>.json` first.
* Existing hostnames are never repointed, moved between tunnels, or their DNS records
  replaced, without `--force`.
* Deploy keys are read-only and per app. With `build=local` the server gets no key and no code.
* Deploying from a branch other than the target never pushes it.
* An existing non-git `~/apps/<name>` is moved aside (`.pre-hl.<ts>`), never deleted.
* Only `down --volumes` deletes data. `copy-data` never writes into a non-empty destination
  without `--force`.

## License

MIT
