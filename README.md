# containerized-dev

Claude Code plugin for containerized multi-workspace development environments.

## What it does

- Scaffolds Docker-based dev environments for any project type (Node.js, Rust, Python, Go, Ruby)
- Global port allocation across all projects (no conflicts)
- Git worktree-based workspaces with isolated containers
- Spawn headless Claude agents in containers, attach interactively later
- **Session handoff** — transfer work context from host to container via `/handoff`
- Host daemon for bridging host-only capabilities (native builds, credential sync)
- Project manifest system with semantic drift detection and self-improvement protocol

## Install

### From the marketplace

```bash
claude /plugin marketplace add tpm/claude-containerized-dev
claude /plugin install containerized-dev
```

### Auto-install via settings

Add to `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": [
    "tpm/claude-containerized-dev"
  ]
}
```

Then install:

```bash
claude /plugin install containerized-dev
```

## Usage

Once installed, the skill activates when you ask Claude to:

- Set up a dev container for your project
- Manage workspaces or worktrees
- Spawn parallel Claude agents in containers
- Hand off work to a containerized agent (`/handoff`)
- Troubleshoot your `dev.sh` setup

### Handoff

From any host Claude session, type `/handoff` to transfer your current work to a containerized agent. The agent writes an environment-agnostic brief, opens a tmux pane (or terminal tab), and starts the container session automatically.

### Project sync

The skill maintains a project manifest. When you update the skill, run a sync to bring projects up to date:

> "Sync my containerized-dev setup"

This checks all manifest resources for drift and regenerates stale files while preserving project-specific customizations.

## Self-improvement

The skill is the single source of truth. Projects never implement containerized-dev features by modifying their own `dev/` resources directly. Instead:

1. Improve the skill (update the plugin)
2. Sync the project
3. Verify

This ensures improvements are available to all projects.
