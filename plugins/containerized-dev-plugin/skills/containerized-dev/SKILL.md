---
name: containerized-dev
description: Set up and manage containerized multi-workspace development environments with Docker. Handles global port allocation (no conflicts across projects), git worktrees, Claude agent spawning, host daemon for container→host services (native builds, credential sync), and project-specific container configuration. Use when user wants to set up dev containers, manage workspaces, spawn parallel Claude agents in containers, trigger host-native builds from containers, or troubleshoot their dev.sh setup.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent
---

# Containerized Dev Environments

Docker-based development with multi-workspace support, global port allocation, and Claude agent integration. Works for any project type (Node.js, Rust, Python, Go, etc.).

## Skill Resources

This skill bundles scripts in its `resources/` directory (relative to this SKILL.md):

| Resource | Purpose |
|----------|---------|
| `resources/credential-sync` | Host-side watcher that syncs `~/.claude/mnt/credentials/.credentials.json` ↔ macOS Keychain |

These are installed to the host during project initialization (see Installation section).

## Host Prerequisites

These must be installed on the macOS host before initializing a project. The skill checks and prompts for missing prerequisites during `init` and `sync`.

| Dependency | Install | Purpose |
|------------|---------|---------|
| Docker Desktop | `brew install --cask docker` | Container runtime |
| Node.js | `brew install node` | Host daemon, Claude Code |

### Installation (run by skill during init)

```bash
install_skill_resources() {
  local skill_dir
  # Resolve the skill's resources directory (relative to where the skill is installed)
  skill_dir="$(find ~/.claude/skills ~/.claude/plugins -name credential-sync -path '*/containerized-dev/resources/*' -print -quit 2>/dev/null)"
  skill_dir="$(dirname "$skill_dir")"

  if [[ -z "$skill_dir" ]] || [[ ! -d "$skill_dir" ]]; then
    echo "Error: Could not find containerized-dev skill resources" >&2
    return 1
  fi

  # ── Check prerequisites (hard stop on missing) ──
  local missing=()
  command -v docker &>/dev/null || missing+=("Docker Desktop: brew install --cask docker")
  command -v node &>/dev/null || missing+=("Node.js: brew install node")

  if [[ ${#missing[@]} -gt 0 ]]; then
    echo "Missing prerequisites:" >&2
    printf "  - %s\n" "${missing[@]}" >&2
    return 1
  fi

  # ── Install credential-sync ──
  local dest="$HOME/.local/bin/credential-sync"
  mkdir -p "$(dirname "$dest")"
  cp "$skill_dir/credential-sync" "$dest"
  chmod +x "$dest"
  echo "Installed credential-sync to $dest" >&2

  # ── Ensure shared file directory exists ──
  mkdir -p "$HOME/.claude/mnt/credentials"
}
```

This is called during `cmd_init()` in dev.sh and during `sync` when the skill detects the resource is missing or outdated.

## Core Concepts

- **Workspace**: An isolated dev environment. `root` = main repo, others = git worktrees.
- **Port registry**: Global JSON file prevents port conflicts across ALL projects on the system.
- **dev.sh**: Per-project CLI that manages containers, worktrees, and Claude agents.
- **Daemon**: Node.js HTTP server on the host, reachable from containers via `host.docker.internal`. Bridges host-only capabilities (native builds, worktree management) to containerized agents.
- **Spawn/attach**: Run headless Claude agents in worktree containers, attach interactively later.

## Global State

**Location**: `~/.local/share/dev-workspaces/`

| File | Purpose |
|------|---------|
| `ports.json` | Port registry — prevents conflicts across all projects |
| `Caddyfile` | Root Caddyfile — static config (CORS snippet, etc.) + `import caddy-sites` |
| `caddy-sites` | Generated Caddy site blocks from all projects/workspaces |

All keys use `{project}:{workspace}` format.

## Global Port Registry

**Location**: `~/.local/share/dev-workspaces/ports.json`

Keys use `{project}:{workspace}` format. Port range: **5173–5999**.

```json
{
  "doess:root": 5173,
  "doess:fix-auth": 5174,
  "my-rust-app:root": 5175
}
```

Port allocation implementation (used in dev.sh):

```bash
GLOBAL_PORTS_FILE="$HOME/.local/share/dev-workspaces/ports.json"
PORT_RANGE_START=5173
PORT_RANGE_END=5999

allocate_port() {
  local key="$1"  # project:workspace
  mkdir -p "$(dirname "$GLOBAL_PORTS_FILE")"
  [[ -f "$GLOBAL_PORTS_FILE" ]] || echo '{}' > "$GLOBAL_PORTS_FILE"
  python3 -c "
import json
d = json.load(open('$GLOBAL_PORTS_FILE'))
if '$key' in d:
    print(d['$key'])
else:
    used = set(d.values())
    for p in range($PORT_RANGE_START, $PORT_RANGE_END + 1):
        if p not in used:
            d['$key'] = p
            json.dump(d, open('$GLOBAL_PORTS_FILE', 'w'), indent=2)
            print(p)
            break
    else:
        raise SystemExit('No free ports')
"
}

free_port() {
  local key="$1"
  [[ -f "$GLOBAL_PORTS_FILE" ]] || return 0
  python3 -c "
import json
d = json.load(open('$GLOBAL_PORTS_FILE'))
d.pop('$key', None)
json.dump(d, open('$GLOBAL_PORTS_FILE', 'w'), indent=2)
"
}
```

## Project Detection

Before scaffolding, detect from the repo root:

| File | Type | Base Image | Dep Dir |
|------|------|------------|---------|
| `package.json` | Node.js | `node:23-bookworm` | `node_modules` |
| `Cargo.toml` | Rust | `rust:1-bookworm` | `target` |
| `pyproject.toml` | Python | `python:3.13-bookworm` | `.venv` |
| `go.mod` | Go | `golang:1.24-bookworm` | — |
| `Gemfile` | Ruby | `ruby:3.4-bookworm` | `vendor/bundle` |

For Rust GUI apps (Bevy, Tauri, GTK, etc.), add system deps. Bevy on Linux needs: `libasound2-dev libudev-dev libwayland-dev libxkbcommon-dev libvulkan-dev libx11-dev libxrandr-dev libxi-dev`. Tauri/GTK needs: `libgtk-3-dev libwebkit2gtk-4.1-dev libssl-dev pkg-config`.

When multiple manifest files are detected (e.g., `Cargo.toml` + `package.json`), it's a **hybrid project**. Use the primary language's official image as base and install the secondary toolchain. See the Dependency Caching section for details.

## Scaffolded Files

Generate a `dev/` directory at the repo root plus `dev.sh`:

```
dev.sh                    # CLI entrypoint (chmod +x)
dev/
  docker-compose.yml      # Parameterized compose file
  Dockerfile.base         # Base image with system deps + claude
  Dockerfile              # Project image (deps pre-installed)
  entrypoint.sh           # Container entrypoint (chmod +x, handles Claude state)
  daemon.cjs              # Host daemon (Node.js HTTP server)
  claude-sharing.json     # Declares what host Claude state containers can access
  skills/
    container-env/
      SKILL.md            # Generated skill teaching container agents their environment
.claude/
  commands/
    handoff.md            # /handoff slash command for host agents
```

### dev.sh Commands

| Command | Description |
|---------|-------------|
| `up [--workspace NAME]` | Allocate port, start container |
| `down [--workspace NAME]` | Stop container |
| `shell [--workspace NAME]` | Shell into container |
| `claude [--workspace NAME]` | Interactive Claude session |
| `server [--workspace NAME]` | Run dev server (web projects) |
| `spawn NAME [prompt] [base]` | Worktree + container + optional headless agent |
| `attach NAME` | Kill headless agent, attach with `--continue` |
| `status` | All workspaces: port, status, branch, last commit |
| `release [CRATE]` | Cross-platform build: host-native + container, output to `dist/` |
| `worktree ls\|add\|rm` | Manage git worktrees |
| `daemon start\|stop\|status\|logs` | Manage the host daemon process |
| `caddy register\|deregister` | Register/deregister Caddy site blocks (web projects with custom domains) |
| `handoff SHORTCODE` | Start container claude with handoff context |
| `handoffs [list\|clean]` | List handoff status or clean completed |
| `build` | Build dev Docker image |
| `base` | Build base Docker image |

### dev.sh Structure

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$SCRIPT_DIR"
PROJECT_NAME="$(basename "$REPO_ROOT")"  # used for container names + port keys
WORKTREES_DIR="$REPO_ROOT/.worktrees"
GLOBAL_PORTS_FILE="$HOME/.local/share/dev-workspaces/ports.json"
PORT_RANGE_START=5173
PORT_RANGE_END=5999
WORKSPACE="root"
```

Key patterns in dev.sh:
- **Container naming**: `{PROJECT_NAME}-{WORKSPACE}` (e.g., `myapp-root`)
- **Compose project name**: `{PROJECT_NAME}-{WORKSPACE}` for isolation
- **Port keys**: `{PROJECT_NAME}:{WORKSPACE}` in global registry
- **Working dir in container**: `/repo` for root, `/repo/.worktrees/{name}` for worktrees
- Parse `--workspace NAME` and `--env NAME` as global flags before command dispatch

### docker-compose.yml Template

```yaml
services:
  dev:
    image: ${PROJECT_NAME}-dev:latest
    build:
      context: ..
      dockerfile: dev/Dockerfile
    container_name: ${PROJECT_NAME}-${WORKSPACE:-root}
    working_dir: ${WORKING_DIR:-/repo}
    volumes:
      # Full repo mount (all worktrees accessible)
      - ..:/repo
      # Anonymous volume for deps (keeps image layer, not overwritten by mount)
      - /repo/${DEP_DIR}
      # Claude: container's own persistent state + host config read-only
      - claude-state:/root/.claude
      - ${HOST_HOME:-~}/.claude:/host-claude:ro
      # Shared bind-mount — all containers see the same credential file instantly via virtiofs
      - ${HOST_HOME:-~}/.claude/mnt/credentials/.credentials.json:/root/.claude/.credentials.json
    ports:
      - "${HOST_PORT:-5173}:${DEV_PORT:-5173}"
    environment:
      - DEV_PORT=${DEV_PORT:-5173}
      - WORKSPACE=${WORKSPACE:-root}
    extra_hosts:
      - "host.docker.internal:host-gateway"
    stdin_open: true
    tty: true

volumes:
  claude-state:   # persistent Claude state for container agents
```

Replace `${DEP_DIR}` with the actual dependency directory for the project type (e.g., `node_modules`, `target`). If none, omit that volume line. Add language-specific cache volumes to the `volumes:` section (see Dependency Caching).

**Additional volume mounts** — add per project as needed:
- Secret managers: `~/.infisical:/root/.infisical:ro`, `~/.aws:/root/.aws:ro`, etc.
- SSH keys: `~/.ssh:/root/.ssh:ro`
- GPG: `~/.gnupg:/root/.gnupg:ro`

### Dependency Caching

Package managers download registries/indexes on every container run unless cached. Use **named Docker volumes** to persist caches across container restarts.

**Named volumes per project type** — add to `volumes:` in services AND declare at top level:

| Type | Cache Volumes | Mount Points |
|------|--------------|--------------|
| Rust | `cargo-registry`, `cargo-git` | `${CARGO_HOME}/registry`, `${CARGO_HOME}/git` (see note) |
| Node.js | `npm-cache` | `/root/.npm` |
| Python | `pip-cache` | `/root/.cache/pip` |
| Go | `go-mod-cache` | `/root/go/pkg/mod` |
| Ruby | `bundle-cache` | `/usr/local/bundle/cache` |

**Rust CARGO_HOME gotcha:** Official `rust:*` images set `CARGO_HOME=/usr/local/cargo`, NOT `~/.cargo`. Images that install Rust via rustup (e.g., `node:*` + rustup) use `~/.cargo`. Always check `CARGO_HOME` in the base image and mount volumes to match.

Example for a Rust project (using `rust:1-bookworm` where `CARGO_HOME=/usr/local/cargo`):

```yaml
services:
  dev:
    volumes:
      - ..:/repo
      - /repo/target            # anonymous volume for build artifacts
      - cargo-registry:/usr/local/cargo/registry
      - cargo-git:/usr/local/cargo/git

volumes:
  cargo-registry:
  cargo-git:
```

**Hybrid projects** (e.g., Rust + Node.js): add cache volumes for ALL languages present. For the base image, pick the primary language's official image and install the secondary toolchain:

| Primary | Secondary | Base Image | Install |
|---------|-----------|------------|---------|
| Rust + Node | Rust | `rust:1-bookworm` | `curl -fsSL https://deb.nodesource.com/setup_23.x \| bash - && apt-get install -y nodejs` |
| Node + Rust | Node | `node:23-bookworm` | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh -s -- -y` |
| Python + Node | Python | `python:3.13-bookworm` | Install Node via nodesource |

Choose primary based on which language has the heavier build toolchain.

**Why named volumes over anonymous volumes for caches:** Anonymous volumes (`- /root/.cargo/registry`) get recreated per compose project name, so worktree containers don't share cache with root. Named volumes (`- cargo-registry:/root/.cargo/registry`) are shared across all containers in the compose project and survive `docker compose down`.

**Pre-warming caches in Dockerfile** — for Rust workspaces, copy manifests + create stub sources + `cargo fetch`:

```dockerfile
COPY Cargo.toml Cargo.lock ./
COPY crates/*/Cargo.toml crates/
# Create stub lib.rs for each crate so cargo can resolve the workspace
RUN find crates -name Cargo.toml -exec sh -c \
    'mkdir -p "$(dirname "$1")/src" && touch "$(dirname "$1")/src/lib.rs"' _ {} \; \
    && cargo fetch \
    && rm -rf crates/*/src
```

This warms the image layer cache. At runtime, the named volume mount provides the actual persistent cache.

### Dockerfile.base Template

Always install Claude Code. Add project-type-specific system deps.

```dockerfile
FROM {base_image}

RUN apt-get update && apt-get install -y --no-install-recommends \
    git curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Project-type-specific system deps here (GUI libs, etc.)

# Install Claude Code globally
# For Node.js projects, npm is already available:
RUN npm install -g @anthropic-ai/claude-code
# For non-Node projects, install Node first:
# RUN curl -fsSL https://deb.nodesource.com/setup_23.x | bash - \
#     && apt-get install -y nodejs \
#     && npm install -g @anthropic-ai/claude-code
```

### Dockerfile Template

```dockerfile
FROM {project}-base:latest

WORKDIR /repo

# Pre-install deps (layer cache)
COPY {dep_files} ./
RUN {install_cmd}

COPY . .

COPY dev/entrypoint.sh /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 5173-5999

CMD ["bash"]
```

Dep files and install commands by type:

| Type | Copy | Install |
|------|------|---------|
| Node.js | `package.json package-lock.json` | `npm ci` |
| Rust | `Cargo.toml Cargo.lock` | `cargo fetch` |
| Python | `pyproject.toml uv.lock` (or `requirements.txt`) | `uv sync` (or `pip install -r requirements.txt`) |
| Go | `go.mod go.sum` | `go mod download` |

### entrypoint.sh Template

```bash
#!/usr/bin/env bash
set -euo pipefail

# Add project bin dirs to PATH for non-interactive shells
# e.g., export PATH="/repo/node_modules/.bin:$PATH"

# Source secrets if an env file exists (populated by secret manager)
if [[ -f /root/.env.docker ]]; then
  source /root/.env.docker
fi

# ── Claude state setup ───────────────────────────────────────────────────────
# Container gets its own persistent state at /root/.claude (named volume).
# Host Claude config is read-only at /host-claude.
# dev/claude-sharing.json declares what to share.

HOST_CLAUDE="/host-claude"
CLAUDE_DIR="/root/.claude"
SHARING_CONFIG="/repo/dev/claude-sharing.json"

setup_claude_state() {
  mkdir -p "$CLAUDE_DIR/skills" "$CLAUDE_DIR/commands"

  if [[ ! -d "$HOST_CLAUDE" ]] || [[ ! -f "$SHARING_CONFIG" ]]; then
    install_container_skill
    return
  fi

  # Parse config
  eval "$(python3 -c "
import json
c = json.load(open('$SHARING_CONFIG'))
print(f'share_creds={str(c.get(\"credentials\", True)).lower()}')
print(f'share_cmds={str(c.get(\"commands\", True)).lower()}')
s = c.get('skills', {})
share = s.get('share', 'none')
if isinstance(share, list):
    print(f'skill_mode=list')
    print(f'skill_list={\" \".join(share)}')
else:
    print(f'skill_mode={share}')
    print(f'skill_list=')
print(f'skill_excludes={\" \".join(s.get(\"exclude\", []))}')
" 2>/dev/null)" || return

  # Credentials — shared bind-mount provides ~/.claude/.credentials.json from host.
  # Docker creates the file as empty if the host path doesn't exist yet; use -s (not -f) to detect that.
  if [[ "\$share_creds" == "true" ]]; then
    if [[ ! -s "\$CLAUDE_DIR/.credentials.json" ]] && [[ -f "\$HOST_CLAUDE/.credentials.json" ]]; then
      cp -f "\$HOST_CLAUDE/.credentials.json" "\$CLAUDE_DIR/.credentials.json"
    fi
    # Mark Claude as already initialized so it skips the onboarding/login flow
    if [[ -f /root/.claude.json ]]; then
      python3 -c "
import json
with open('/root/.claude.json') as f: d = json.load(f)
d['hasCompletedOnboarding'] = True
with open('/root/.claude.json', 'w') as f: json.dump(d, f, indent=2)
"
    else
      echo '{"hasCompletedOnboarding":true}' > /root/.claude.json
    fi
  fi

  # Wrap claude to skip permissions (containers are already sandboxed)
  # IS_SANDBOX=1 tells Claude Code to allow --dangerously-skip-permissions as root
  if [[ -L /usr/local/bin/claude ]]; then
    CLAUDE_REAL="\$(readlink -f /usr/local/bin/claude)"
    rm /usr/local/bin/claude
    printf '#!/bin/bash\nexport IS_SANDBOX=1\nexec "%s" --dangerously-skip-permissions "\$@"\n' "\$CLAUDE_REAL" > /usr/local/bin/claude
    chmod +x /usr/local/bin/claude
  fi

  # Commands — symlink each file individually
  if [[ "\$share_cmds" == "true" ]] && [[ -d "\$HOST_CLAUDE/commands" ]]; then
    for f in "\$HOST_CLAUDE/commands"/*; do
      [[ -f "\$f" ]] && ln -sf "\$f" "\$CLAUDE_DIR/commands/\$(basename "\$f")"
    done
  fi

  # Skills — check include/exclude
  if [[ "\$skill_mode" != "none" ]] && [[ -d "\$HOST_CLAUDE/skills" ]]; then
    local excludes=" \$skill_excludes "
    for d in "\$HOST_CLAUDE/skills"/*/; do
      [[ -d "\$d" ]] || continue
      local name="\$(basename "\$d")"
      [[ "\$excludes" == *" \$name "* ]] && continue
      if [[ "\$skill_mode" == "all" ]] || [[ " \$skill_list " == *" \$name "* ]]; then
        ln -sf "\$d" "\$CLAUDE_DIR/skills/\$name"
      fi
    done
  fi

  install_container_skill
}

install_container_skill() {
  local src="/repo/dev/skills/container-env"
  if [[ -d "\$src" ]]; then
    rm -rf "\$CLAUDE_DIR/skills/container-env"
    cp -r "\$src" "\$CLAUDE_DIR/skills/container-env"
  fi
}

setup_claude_state

exec "$@"
```

## Git Worktree Management

### Creating Worktrees

```bash
worktree_ensure() {
  local name="$1" base="${2:-HEAD}"
  local wt_dir="$WORKTREES_DIR/$name"
  [[ -d "$wt_dir" ]] && return 1

  mkdir -p "$WORKTREES_DIR"
  local branch="wt/$name"
  if git rev-parse --verify "$branch" &>/dev/null; then
    git worktree add "$wt_dir" "$branch"
  else
    git worktree add -b "$branch" "$wt_dir" "$base"
  fi

  # CRITICAL: Rewrite git paths to relative so they work inside containers
  echo "gitdir: ../../.git/worktrees/$name" > "$wt_dir/.git"
  echo "../../../.worktrees/$name" > "$REPO_ROOT/.git/worktrees/$name/gitdir"
}
```

The relative-path rewrite is essential — without it, git inside the container sees absolute host paths that don't exist.

### Removing Worktrees

Stop container first, then remove worktree, branch, and free the port.

## Claude Agent Integration

### Claude State Management

Containers get their **own persistent Claude state** via a named Docker volume (`claude-state`), separate from the host's `~/.claude`. The host's Claude config is mounted read-only at `/host-claude`. On container startup, the entrypoint reads `dev/claude-sharing.json` and symlinks declared items into the container's `/root/.claude/`.

**Permissions**: The entrypoint wraps the `claude` binary to always pass `--dangerously-skip-permissions` with `IS_SANDBOX=1`. Containers run as root by default, and Claude Code rejects `--dangerously-skip-permissions` as root unless `IS_SANDBOX=1` is set. Containers are already sandboxed — permission prompts add friction with no security benefit. This means `claude` just works in any container context (interactive, headless, handoff) with no permission prompts.

**Onboarding**: The entrypoint sets `hasCompletedOnboarding: true` in `/root/.claude.json` so Claude skips the first-run login flow.

**`dev/claude-sharing.json`** — declares what host Claude state to share:

```json
{
  "credentials": true,
  "commands": true,
  "skills": {
    "share": "all",
    "exclude": ["containerized-dev"]
  }
}
```

- `credentials`: Share `~/.claude/.credentials.json` (access tokens). Almost always `true`.
- `commands`: Share `~/.claude/commands/` (user slash commands). Almost always `true`.
- `skills.share`: `"all"`, `"none"`, or `["skill-name-1", "skill-name-2"]`.
- `skills.exclude`: Always excluded even when `share` is `"all"`. The `containerized-dev` skill itself should always be excluded — it manages the host environment, not the container.

This file is committed to git alongside the rest of `dev/`.

### Container-env Skill

Each project gets a generated `dev/skills/container-env/SKILL.md` that teaches container agents about their environment. This is **not a generic template** — it's tailored to the project type during scaffolding and kept in sync when the dev environment changes.

The container-env skill should cover:
- What platform the container runs (Linux) and what the host is (macOS, etc.)
- Project layout and what's mounted where
- What can be built/run locally vs what requires the host
- Available host services (daemon endpoints, helper scripts like `host-build`)
- Constraints (don't run Docker, don't run dev.sh, don't touch host build caches)
- Networking (how to reach the host, other containers/services)

**Project-type examples** — the container-env content varies by project:

| Project Type | Key Container-env Content |
|-------------|--------------------------|
| Rust GUI (Bevy/Metal) | Use `host-build` for macOS builds, `cargo build` works for libs, can't run GPU app |
| Web app (Node/React) | Dev server runs in container, accessible at allocated port, hot reload works |
| Web + Supabase | Supabase container at `supabase:5432`, don't install Postgres, use service URL |
| Python ML | GPU not available in container, use host for CUDA builds, CPU inference works |
| Go + external APIs | Mock services at `host.docker.internal:{port}`, don't modify docker-compose |

### Credential Sync (macOS) — Shared Bind-Mount

Container Claude sessions share OAuth tokens. When one session refreshes a token, the old token is invalidated — all other sessions must see the new token immediately. A **shared bind-mount** of a single file on the host gives all containers instant access to the same credentials via Docker's virtiofs. A lightweight host-side watcher keeps the macOS Keychain in sync (for the host's benefit — containers don't care about the Keychain).

**Why**: Without this, token refreshes in one container don't propagate to others or back to the host. Sessions go stale and get logged out. With a shared file, all containers see changes instantly (zero delay via virtiofs). The Keychain watcher is async — it's only needed so host-side Claude sessions also stay current.

#### Architecture

```
macOS Keychain
    ↕ (credential-sync watcher, async)
~/.claude/mnt/credentials/.credentials.json  (regular file on host)
    ├── bind-mount → Container A: /root/.claude/.credentials.json
    ├── bind-mount → Container B: /root/.claude/.credentials.json
    └── bind-mount → Container C: /root/.claude/.credentials.json
```

- **Inter-container sync**: Instant. Same file via virtiofs. Zero delay.
- **Keychain sync**: Async watcher on host. For host Claude's benefit only — containers don't care.

#### Credential Sync Script: `credential-sync`

**Source**: `resources/credential-sync` (bundled with this skill)
**Installed to**: `~/.local/bin/credential-sync` (by `install_skill_resources()` during init)

Pure bash script. No Python, no FUSE, no system extensions.

**Interface:**
```
credential-sync seed              # Keychain → shared file (one-shot)
credential-sync start [--fg]      # seed + start watcher (daemonizes by default)
credential-sync stop              # kill watcher
credential-sync status            # running or not
```

**Watcher**: Polls with `stat -f %m` every 2 seconds. On change → updates Keychain via `security delete-generic-password` + `security add-generic-password`. On startup → seeds shared file from Keychain (falls back to `~/.claude/.credentials.json`).

**PID file**: `~/.claude/mnt/.credential-sync.pid`

Do **not** inline or duplicate this script in project resources. It's installed from the skill's `resources/` directory to `~/.local/bin/` and shared across all projects.

#### Credential Lifecycle in dev.sh

```bash
CRED_SHARED_DIR="$HOME/.claude/mnt/credentials"
CRED_SHARED_FILE="$CRED_SHARED_DIR/.credentials.json"

credential_ensure() {
  # Make sure the shared file exists so Docker doesn't create it as a directory
  mkdir -p "$CRED_SHARED_DIR"
  if [[ ! -f "$CRED_SHARED_FILE" ]]; then
    touch "$CRED_SHARED_FILE"
    chmod 600 "$CRED_SHARED_FILE"
  fi

  # Start the keychain sync watcher if credential-sync is installed
  if command -v credential-sync &>/dev/null; then
    credential-sync start
  else
    echo "Warning: credential-sync not installed. Keychain won't stay in sync." >&2
    echo "  Containers still share credentials via bind-mount." >&2
  fi
}

credential_stop() {
  if command -v credential-sync &>/dev/null; then
    credential-sync stop
  fi
}
```

`credential_ensure` is called from `cmd_up()` before `compose up`. It's idempotent — all projects share the same global file. If `credential-sync` isn't installed, containers still share the bind-mounted file for inter-container sync; only Keychain sync is lost.

#### Fallback: No credential-sync

If `credential-sync` isn't installed, the system degrades gracefully:
1. `credential_ensure()` still creates the shared file directory and touches the file
2. The entrypoint detects an empty file (`! -s`) and copies credentials from the host read-only mount
3. Inter-container sync still works (same bind-mounted file)
4. Only Keychain sync is lost — the host's Keychain won't be updated when containers refresh tokens

### Spawning Headless Agents

```bash
docker exec -d "{project}-{name}" \
  bash -c "claude -p '$escaped_prompt' \
    > /repo/.worktrees/$name/.claude-agent.log 2>&1"
```

The container's claude wrapper automatically includes `--dangerously-skip-permissions` — no need to pass it explicitly.

### Attaching to Running Agents

1. Kill headless: `docker exec "{project}-{name}" pkill -f "claude.*-p"`
2. Show context: `git log --oneline -5` + `git diff --stat`
3. Attach: `docker compose exec dev claude --continue`

## Daemon (daemon.cjs)

The daemon is a Node.js HTTP server that runs on the **host** and provides services to containers. It **must listen on `0.0.0.0`** (not `127.0.0.1`) so containers can reach it via `host.docker.internal:{DAEMON_PORT}`.

```javascript
server.listen(PORT, "0.0.0.0", () => { ... });
```

**Standard daemon port**: 7701 (outside the workspace port range).

### Core Daemon Endpoints

| Endpoint | Description |
|----------|-------------|
| `POST /spawn` | Create workspace + start container + optional headless agent |
| `GET /status` | List all workspaces with status |
| `GET /worktree/:name/activity` | Git log, status, agent log tail |
| `DELETE /worktree/:name` | Stop container + remove workspace (also deregisters Caddy site block) |
| `POST /caddy/register` | Register Caddy site block for workspace (web projects with custom domains) |
| `POST /caddy/deregister` | Remove Caddy site block for workspace |
| `GET /caddy/sites` | List registered Caddy site blocks |

### Daemon Management in dev.sh

```bash
DAEMON_PORT=7701
DAEMON_PID_FILE="$WORKTREES_DIR/.daemon.pid"

daemon_start() {
  if daemon_running; then return; fi
  nohup node "$REPO_ROOT/dev/daemon.cjs" > "$WORKTREES_DIR/.daemon.log" 2>&1 &
  echo "$!" > "$DAEMON_PID_FILE"
}
```

The daemon auto-starts on `./dev.sh up` and stores its PID in `.worktrees/.daemon.pid`.

### Container → Host Communication

Containers reach the host daemon via Docker's built-in DNS:

```
http://host.docker.internal:7701/endpoint
```

This requires `extra_hosts: ["host.docker.internal:host-gateway"]` in docker-compose.yml (already in the template).

## Host Build Service (Optional)

For projects where the container can't build certain targets (e.g., macOS Metal/GPU apps, platform-specific binaries), the daemon can proxy builds to the host.

### When to Add

- Rust GUI apps using Metal, Vulkan, or platform-specific graphics APIs
- Projects that need native macOS/Windows binaries but develop in Linux containers
- Any crate/package that requires host-only SDKs or toolchains

### Daemon Endpoints

**`POST /build`** — Synchronous NDJSON streaming response:
- Request: `{"crate": "my-app", "profile": "release"}`
- Runs build command on host with a separate target directory (`target-host/`) to prevent toolchain corruption
- Streams NDJSON: `{type: "started"}`, `{type: "stderr", data: "..."}`, `{type: "complete", success: true, artifact: "dist/..."}`
- On success, copies binary to `dist/{host_target_triple}/{binary}`
- Single-build mutex — returns **409** if a build is already in progress
- Kills the build process if the client disconnects (no orphaned builds)

**`GET /build/targets`** — Discovery: `{host_target, binaries: [...], busy: bool}`

**`GET /build/status`** — Whether a build is currently running.

### Separate Target Directories

Host and container builds **must** use different target directories. Mixing toolchain metadata corrupts incremental compilation state.

```
target/          # Container builds (Linux, gitignored)
target-host/     # Host builds (macOS/Windows, gitignored)
```

Both directories should be in `.gitignore`.

### Container Helper Script

Install a small CLI wrapper in the container that curls the daemon:

```dockerfile
COPY dev/host-build /usr/local/bin/host-build
RUN chmod +x /usr/local/bin/host-build
```

Usage from inside a container:
```bash
host-build                    # Show available targets
host-build my-app             # Build with default profile (release)
host-build my-app --profile dev
```

The script streams NDJSON from the daemon, prints build output in real time, and exits 0/1 matching build success/failure.

### Artifact Layout

Cross-platform artifacts go in `dist/` organized by target triple:

```
dist/
  aarch64-apple-darwin/
    my-app
  x86_64-unknown-linux-gnu/
    my-app
```

### Release Command (dev.sh)

`./dev.sh release [TARGET]` orchestrates both platforms from the host side:
- **Host-native build**: Runs the build command directly (no daemon roundtrip when already on the host)
- **Container build**: Runs via `docker exec ... cargo build --release` (or equivalent)
- Copies artifacts to `dist/{target_triple}/`
- Auto-discovers buildable targets if none specified

## Handoff System

Handoffs transfer work context from a host Claude Code session to a containerized agent. The host agent writes an environment-agnostic brief; the container agent picks it up and works autonomously.

### Handoff File Format

Files live at `.claude/handoffs/{shortcode}.md`. The shortcode is a descriptive slug derived from the current work (e.g., `ts-errors`, `stripe-drift`, `auth-refactor`).

```markdown
---
shortcode: ts-errors
status: pending
host_session: 46708c87-487d-47f0-9ea0-f3acf08ddf50
container_session:
---

## Context

[Environment-agnostic summary of work state]

### What was decided and why
- ...

### What's been done
- ...

### What remains
- ...

### Key findings / gotchas
- ...

### Branch
feature-branch-name
```

**Status lifecycle**: `pending` → `active` → `completed`

**What goes in the brief**: decisions, rationale, commits made, remaining work, gotchas discovered. All paths relative to repo root. No host-specific paths, tool outputs, or environment details.

**What does NOT transfer**: raw transcript, host paths, permission history, stale tool outputs.

### Session Self-Discovery Protocol

Container agents need their own session ID but Claude Code doesn't expose it natively. The agent discovers it by fingerprinting its own transcript:

1. Echo a unique marker — it gets recorded in the session's JSONL file
2. Search session storage for the file containing that marker
3. Extract the session ID from the filename

```bash
MARKER="CLSESS-$(uuidgen)" && echo "$MARKER" && sleep 1 && \
SESSION_FILE=$(grep -rl "$MARKER" ~/.claude/projects/ --include="*.jsonl" 2>/dev/null | head -1) && \
echo "SESSION_ID=$(basename "$SESSION_FILE" .jsonl)"
```

This must be included in every container-env skill so containerized agents can self-identify.

### dev.sh Handoff Commands

Add these to dev.sh:

| Command | Description |
|---------|-------------|
| `handoff SHORTCODE` | Start container claude with handoff context |
| `handoffs` | List all handoffs with status |
| `handoffs clean` | Remove completed handoffs |

#### `cmd_handoff` implementation

```bash
cmd_handoff() {
  local shortcode="${1:-}"
  local handoff_dir="$REPO_ROOT/.claude/handoffs"

  if [[ -z "$shortcode" ]]; then
    echo "Usage: ./dev.sh handoff SHORTCODE" >&2
    echo "" >&2
    echo "Available handoffs:" >&2
    cmd_handoffs
    exit 1
  fi

  local handoff_file="$handoff_dir/$shortcode.md"
  if [[ ! -f "$handoff_file" ]]; then
    echo "Error: No handoff file at $handoff_file" >&2
    cmd_handoffs
    exit 1
  fi

  local status
  status="$(sed -n 's/^status: *//p' "$handoff_file")"
  if [[ "$status" != "pending" ]]; then
    echo "Error: Handoff '$shortcode' has status '$status' (expected 'pending')." >&2
    exit 1
  fi

  validate_name "$WORKSPACE"
  compose_env "$WORKSPACE"

  if ! is_running "$WORKSPACE"; then
    echo "Container not running. Starting..." >&2
    cmd_up
  fi

  # Mark active
  sed -i '' "s/^status: pending/status: active/" "$handoff_file" 2>/dev/null || \
    sed -i "s/^status: pending/status: active/" "$handoff_file"

  echo "Starting handoff '$shortcode' in container ${PROJECT_NAME}-$WORKSPACE..."
  compose exec dev claude \
    --append-system-prompt-file "/repo/.claude/handoffs/$shortcode.md"

  # After claude exits, check if the agent marked it completed
  status="$(sed -n 's/^status: *//p' "$handoff_file")"
  if [[ "$status" == "active" ]]; then
    # Agent didn't mark completion — check for container session
    echo "Handoff '$shortcode' session ended (status still active)."
  fi
}
```

#### `cmd_handoffs` implementation

```bash
cmd_handoffs() {
  local subcmd="${1:-list}"
  local handoff_dir="$REPO_ROOT/.claude/handoffs"

  case "$subcmd" in
    list|ls)
      if [[ ! -d "$handoff_dir" ]] || ! ls "$handoff_dir"/*.md &>/dev/null; then
        echo "No handoffs."
        return
      fi
      printf "%-20s %-12s %-40s\n" "SHORTCODE" "STATUS" "CONTAINER SESSION"
      printf "%-20s %-12s %-40s\n" "─────────" "──────" "─────────────────"
      for f in "$handoff_dir"/*.md; do
        local sc st cs
        sc="$(basename "$f" .md)"
        st="$(sed -n 's/^status: *//p' "$f")"
        cs="$(sed -n 's/^container_session: *//p' "$f")"
        printf "%-20s %-12s %-40s\n" "$sc" "$st" "${cs:--}"
      done
      ;;
    clean)
      if [[ ! -d "$handoff_dir" ]]; then return; fi
      local cleaned=0
      for f in "$handoff_dir"/*.md; do
        [[ -f "$f" ]] || continue
        local st
        st="$(sed -n 's/^status: *//p' "$f")"
        if [[ "$st" == "completed" ]]; then
          rm "$f"
          echo "Removed: $(basename "$f" .md)"
          ((cleaned++))
        fi
      done
      echo "$cleaned handoff(s) cleaned."
      ;;
    *)
      echo "Usage: ./dev.sh handoffs [list|clean]" >&2
      ;;
  esac
}
```

### Host Agent's Role in Handoff

When a host Claude agent is asked to hand off work (or decides a task would benefit from containerized execution), it must do all of the following automatically:

**Step 1 — Pick a shortcode.** A lowercase slug describing the work, 2-4 words max, hyphen-separated. Examples: `ts-errors`, `stripe-drift`, `auth-refactor`, `fix-queue-handler`.

**Step 2 — Write the handoff file.** Create `.claude/handoffs/{shortcode}.md` using this exact template:

```markdown
---
shortcode: {shortcode}
status: pending
host_session: {current session ID, or "unknown"}
container_session:
---

## Context

{1-2 sentence summary of what the task is}

### What was decided and why
- {key decisions made, with rationale}

### What's been done
- {commits, file changes, findings — reference by relative path}

### What remains
- {concrete list of remaining work items}

### Key findings / gotchas
- {anything non-obvious the container agent needs to know}

### Branch
{current git branch name}
```

Rules for the brief:
- All file paths relative to repo root (e.g., `app/lib/context.ts`, NOT `/Users/.../context.ts`)
- No host-specific details (no absolute paths, no host env vars, no raw tool outputs)
- Be specific and actionable — the container agent starts fresh with only this context
- Include the git branch so the container agent knows where it's working

**Step 3 — Create the handoffs directory if needed and launch in a new pane.**

```bash
mkdir -p .claude/handoffs
DEVSH="$(git rev-parse --show-toplevel)/dev.sh"

if [ -n "$TMUX" ]; then
  tmux split-window -h "$DEVSH handoff {shortcode}"
elif [ "$TERM_PROGRAM" = "iTerm.app" ]; then
  osascript -e "tell application \"iTerm2\" to tell current window to create tab with default profile command \"$DEVSH handoff {shortcode}\""
else
  osascript -e "tell application \"Terminal\" to do script \"$DEVSH handoff {shortcode}\""
fi
```

**Step 4 — Confirm to the user.** Tell them the handoff shortcode and that the container session is starting in a new pane/tab.

### Container-env Skill: Handoff Section

Every generated container-env skill must include a "Handoff Protocol" section teaching the container agent how to:

1. Discover its own session ID using the self-discovery protocol
2. Claim the handoff (update status to `active`, record `container_session`)
3. Do the work described in the brief
4. Mark the handoff `completed` when done

This section should be included verbatim — it's not project-specific.

## .gitignore Additions

```
.worktrees/
.claude/handoffs/
/target-host    # if using host build service
```

## Project Manifest

Every containerized-dev project has a manifest — the set of resources the skill expects to exist, with content that reflects the skill's current instructions. The manifest is checked during initialization and sync.

**A project is never modified directly for containerized-dev features.** All changes flow through the skill: improve the skill → sync the project → verify.

### Manifest by Project Type

#### All project types (universal)

| Resource | Path | Content governed by |
|----------|------|---------------------|
| Entry script | `dev.sh` | dev.sh Commands + Handoff Commands sections |
| Compose file | `dev/docker-compose.yml` | docker-compose.yml Template section |
| Base image | `dev/Dockerfile.base` | Dockerfile.base Template section |
| Dev image | `dev/Dockerfile` | Dockerfile Template section |
| Entrypoint | `dev/entrypoint.sh` | entrypoint.sh Template section |
| Sharing config | `dev/claude-sharing.json` | Claude State Management section |
| Container-env skill | `dev/skills/container-env/SKILL.md` | Container-env Skill section + project-specific context |
| Handoff command | `.claude/commands/handoff.md` | Handoff Command Template below |
| Gitignore entries | `.gitignore` | `.worktrees/`, `.claude/handoffs/` |

#### Optional resources (added based on project needs)

| Resource | When | Content governed by |
|----------|------|---------------------|
| Host daemon | `dev/daemon.cjs` | When project needs host services (builds, proxy, etc.) |
| Host build helper | `dev/host-build` | When container can't build certain targets |
| Caddy site config | `dev/caddy.json` | When project uses custom local domains (Reverse Proxy section) |
| Credential sync | `~/.local/bin/credential-sync` | Global (shared across projects), Credential Sync section |

### Handoff Command Template

`.claude/commands/handoff.md` — installed in the project so users can invoke `/handoff`:

```markdown
Hand off the current work to a containerized agent. Do ALL of the following:

1. Pick a shortcode: a lowercase hyphen-separated slug (2-4 words) describing the current work. Examples: `ts-errors`, `stripe-drift`, `auth-refactor`.

2. Create the handoff directory and file:

\`\`\`bash
mkdir -p .claude/handoffs
\`\`\`

Write `.claude/handoffs/{shortcode}.md` with this exact format:

\`\`\`markdown
---
shortcode: {shortcode}
status: pending
host_session: {your session ID if known, otherwise "unknown"}
container_session:
---

## Context

{1-2 sentence summary of what the task is}

### What was decided and why
- {key decisions made, with rationale}

### What's been done
- {commits, file changes, findings — reference by relative path from repo root}

### What remains
- {concrete list of remaining work items}

### Key findings / gotchas
- {anything non-obvious the container agent needs to know}

### Branch
{current git branch name}
\`\`\`

Rules for the brief:
- All file paths relative to repo root — NO absolute host paths like /Users/...
- No host-specific details — no env vars, no raw tool outputs, no permission history
- Be specific and actionable — the container agent starts fresh with ONLY this context

3. Launch the handoff in a new pane. Detect the terminal environment and use the right method:

\`\`\`bash
# Get the absolute path to dev.sh (do NOT use relative ./dev.sh)
DEVSH="$(git rev-parse --show-toplevel)/dev.sh"

if [ -n "$TMUX" ]; then
  # tmux — split a new pane
  tmux split-window -h "$DEVSH handoff {shortcode}"
elif [ "$TERM_PROGRAM" = "iTerm.app" ]; then
  osascript -e "tell application \"iTerm2\" to tell current window to create tab with default profile command \"$DEVSH handoff {shortcode}\""
else
  osascript -e "tell application \"Terminal\" to do script \"$DEVSH handoff {shortcode}\""
fi
\`\`\`

4. Tell the user: the handoff shortcode, and that the container session is starting in a new pane/tab.
```

### Manifest Check Semantics

When checking a project against its manifest, the agent does NOT compare file hashes. It evaluates semantically:

For each manifest resource:
1. **Existence**: Does the file exist at the expected path?
2. **Intent alignment**: Does the file's content reflect the current skill's instructions for that resource? Read the skill section that governs the resource, read the project file, and check whether the project file implements what the skill describes.
3. **Completeness**: Are all features described in the governing skill section present? (e.g., does `dev.sh` have `handoff` and `handoffs` commands? Does the entrypoint install the container-env skill?)

Project-specific customizations (extra volumes, env vars, system deps, project-specific container-env content) are preserved — only skill-governed structure and features are checked.

## Initializing a New Project

1. Detect project type from repo root
2. **Ask the user** (if not already evident) about the project's development context:
   - What kind of app? (CLI tool, GUI app, web server, library, etc.)
   - Platform-specific build requirements? (Metal/GPU, native SDKs, etc.)
   - External services? (databases, APIs, other containers)
3. Derive `PROJECT_NAME` from directory name
4. Generate all manifest resources, customized for the project type
5. `chmod +x dev.sh dev/entrypoint.sh`
6. Tell the user to run `./dev.sh base && ./dev.sh up`

## Syncing an Existing Project

When a user asks to update/sync their containerized-dev setup, or when the agent detects drift:

1. Read the manifest for the project type
2. For each resource, run the manifest check (existence + intent alignment + completeness)
3. Report findings as a table:

```
Resource                   Status
────────                   ──────
dev.sh                     OK
dev/entrypoint.sh          DRIFT — missing container-env skill installation
dev/skills/container-env/  DRIFT — missing handoff protocol section
.claude/commands/handoff.md MISSING
```

4. For each MISSING or DRIFT resource, regenerate or patch the file to align with the skill
5. Preserve project-specific customizations when patching
6. Verify by re-running the manifest check — all resources should show OK

## Self-Improvement Protocol

The containerized-dev skill is the single source of truth. Projects are derived instances. **A project never implements new containerized-dev features by modifying its own `dev/` resources.** Instead:

### When a user wants to improve their containerized-dev setup:

1. **Improve the skill first.** Read the current skill, identify what needs to change, and edit `~/.claude/skills/containerized-dev/SKILL.md`. This might mean:
   - Adding a new resource to the manifest
   - Updating a template section
   - Adding a new dev.sh command template
   - Expanding the container-env skill requirements
   - Adding a new section for a new capability

2. **Sync the project.** Run the manifest check against the current project. The changes made in step 1 will show up as DRIFT or MISSING. Regenerate/patch the affected resources.

3. **Verify.** Re-run the manifest check. All resources should show OK. Test the new feature works.

This ensures every improvement is available to ALL projects that use the skill, not just the one where the idea originated.

## Reverse Proxy (Optional, Web Projects Only)

If the project uses custom local domains, the user may want a reverse proxy (Caddy) in front of the dev server. This is project-specific — don't scaffold it unless asked.

### Design: Global Site Registration

There is one Caddy process on the system listening on 443. Site blocks from **all projects and workspaces** are registered into a single shared file, just like the global port registry. Each workspace registers its own block; conflicts (e.g., two workspaces claiming the same domain) cause `caddy reload` to fail, and the error is surfaced to the caller.

### Global Caddy State

**Location**: `~/.local/share/dev-workspaces/` (same directory as `ports.json`)

| File | Purpose |
|------|---------|
| `Caddyfile` | Root Caddyfile — static config + `import caddy-sites` |
| `caddy-sites` | Generated file with per-workspace site blocks (markers delimit each) |

The root `Caddyfile` is created once (on first project that needs Caddy) and rarely changes. The `caddy-sites` file is managed entirely by daemon registrations.

Example root Caddyfile:

```caddy
(cors) {
  @cors_preflight method OPTIONS
  @cors header Origin {args.0}

  handle @cors_preflight {
    header Access-Control-Allow-Origin "{args.0}"
    header Access-Control-Allow-Methods "GET, POST, PUT, PATCH, DELETE"
    header Access-Control-Allow-Headers "Content-Type"
    header Access-Control-Max-Age "3600"
    header Access-Control-Allow-Credentials "true"
    respond "" 204
  }

  handle @cors {
    header Access-Control-Allow-Origin "{args.0}"
    header Access-Control-Expose-Headers "Link"
    header Access-Control-Allow-Credentials "true"
  }
}

# Project-specific static sites can be added here

import caddy-sites
```

### 1. Project-level site config: `dev/caddy.json`

Each project declares what Caddy site blocks it needs. This replaces hardcoded domain constants.

```json
{
  "sites": [
    {
      "domains": [
        "app.myproject.localhost",
        "api.myproject.localhost",
        "webhooks.myproject.localhost"
      ],
      "directives": [
        "import cors https://myproject.localhost"
      ]
    }
  ]
}
```

`domains` — the Caddy site address(es) for this block. `directives` — extra Caddy directives injected into the block (CORS snippets, headers, etc.). The `reverse_proxy` directive is generated automatically from the workspace's allocated port.

### 2. Managed site blocks: `~/.local/share/dev-workspaces/caddy-sites`

Markers use `{project}:{workspace}` — the same key format as the port registry:

```caddy
# containerized-dev:myproject:root:begin
app.myproject.localhost, api.myproject.localhost {
  reverse_proxy localhost:5173
  import cors https://myproject.localhost
}
# containerized-dev:myproject:root:end

# containerized-dev:otherapp:root:begin
otherapp.localhost {
  reverse_proxy localhost:5175
}
# containerized-dev:otherapp:root:end
```

All projects share this file. A workspace from project A and a workspace from project B coexist. If two workspaces (from the same or different projects) claim the same domain, `caddy reload` fails and the registering workspace gets the error.

### 3. Daemon API: Caddy Registration

**Add to `daemon.cjs`**:

```
CADDY_DIR = ~/.local/share/dev-workspaces
CADDY_SITES = CADDY_DIR/caddy-sites
CADDYFILE = CADDY_DIR/Caddyfile
```

- `readCaddyConfig()` — reads `dev/caddy.json` from the project
- `generateSiteBlock(project, workspace, port)` — builds Caddy block text from config + port, wrapped in `# containerized-dev:{project}:{workspace}:begin/end` markers
- `registerSiteBlock(project, workspace, port)` — removes old block for this project:workspace, appends new one, calls `reloadCaddy()`. **On reload failure**: rolls back `caddy-sites` to previous content, rethrows error
- `deregisterSiteBlock(project, workspace)` — removes block by markers, reloads
- `ensureCaddyfile()` — creates root `Caddyfile` + empty `caddy-sites` if they don't exist yet
- `reloadCaddy()` — runs `caddy reload --config CADDYFILE`, captures stderr on failure and throws with the Caddy error message

**New daemon endpoints**:

| Endpoint | Description |
|----------|-------------|
| `POST /caddy/register` `{ workspace }` | Register site block, returns 200 or 409 with error detail |
| `POST /caddy/deregister` `{ workspace }` | Remove site block |
| `GET /caddy/sites` | List all registered blocks across all projects |

The daemon already knows `PROJECT_NAME` from its context. Endpoints accept `{ workspace }` and derive the full `{project}:{workspace}` key internally.

**Modified**: `DELETE /worktree/:name` — calls `deregisterSiteBlock` during cleanup.

### 4. Shell changes: `dev.sh`

**Remove**: `caddy_switch()`

**Add**:

```bash
caddy_register() {
  local ws="$1"
  local resp
  resp=$(curl -s -w "\n%{http_code}" -X POST \
    "http://localhost:$DAEMON_PORT/caddy/register" \
    -H "Content-Type: application/json" \
    -d "{\"workspace\": \"$ws\"}")
  local body status
  body=$(echo "$resp" | head -n -1)
  status=$(echo "$resp" | tail -1)
  if [[ "$status" != "200" ]]; then
    echo "Error: Caddy registration failed for workspace '$ws':" >&2
    echo "$body" >&2
    exit 1
  fi
}

caddy_deregister() {
  local ws="$1"
  curl -s -X POST "http://localhost:$DAEMON_PORT/caddy/deregister" \
    -H "Content-Type: application/json" \
    -d "{\"workspace\": \"$ws\"}" > /dev/null 2>&1 || true
}
```

**Modified functions**:

- `caddy_start()` — starts Caddy with the global Caddyfile (`~/.local/share/dev-workspaces/Caddyfile`) if not already running
- `cmd_up()` — replace `caddy_switch` with `caddy_register`
- `cmd_server()` — add `caddy_start` + `caddy_register` before starting dev server (makes Caddy a prerequisite)
- `cmd_down()` — add `caddy_deregister` after stopping container
- `cmd_caddy()` — replace `switch` subcommand with `register`/`deregister`

### 5. Error surfacing flow

```
cmd_server → caddy_register() → curl POST /caddy/register
  → daemon: registerSiteBlock() → writes caddy-sites → reloadCaddy()
    → caddy reload fails (e.g., duplicate domain across projects)
    → daemon: rolls back caddy-sites, returns 409 { error, detail }
  → caddy_register(): prints error to stderr, exits 1
  → cmd_server: never starts dev server
```

### 6. Cleanup

Workspace removal (any path) calls `deregisterSiteBlock` to remove the block from `caddy-sites` and reload Caddy.

## Extending the Daemon

The daemon is the extension point for any host-only capability containers need. Common patterns:

- **Build services**: Proxy builds requiring host SDKs/toolchains (Metal, Xcode, etc.)
- **Credential sync**: Handled by shared bind-mount + host-side watcher (see Credential Sync section), not the daemon
- **File watching**: Trigger container actions on host filesystem events
- **Reverse proxy management**: Register/deregister Caddy site blocks per workspace

All endpoints should follow existing conventions:
- JSON request/response for simple operations
- NDJSON streaming for long-running operations
- Mutex/409 for operations that can't run concurrently
- Kill child processes on client disconnect
