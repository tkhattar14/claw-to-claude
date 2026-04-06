# claw-to-claude

Migrate an OpenClaw multi-agent setup to Claude Code Agent Teams.

## What This Does

`claw-to-claude` is a Claude Code plugin that reads your existing OpenClaw configuration and workspace files, then generates Claude Code Agent Teams definitions ready to use. It produces persistent memory files, per-agent MCP server configurations, shared skill files, and scheduled task definitions — converting your entire OpenClaw org hierarchy into a set of coordinating CC agents with routing instructions and teammate messaging enabled.

## Prerequisites

- An existing OpenClaw setup (running instance or a snapshot directory)
- Claude Code v2.1.32 or later (Agent Teams requires this version)
- Agent Teams enabled in your Claude Code settings:
  ```json
  { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": true }
  ```

## Installation

```bash
claude plugins install github:onceuponme/claw-to-claude
```

## Usage

**From a local snapshot**

Copy your OpenClaw workspace to your local machine first, then run the migration:

```bash
scp -r user@host:~/openclaw-snapshot ./
```

```
/claw-to-claude:migrate --path ./openclaw-snapshot
```

**Over SSH directly**

```
/claw-to-claude:migrate --ssh user@host
```

## How It Works

Migration runs in three phases:

**Phase 1 — Analyze**
Reads all agent definitions (IDENTITY, SOUL, AGENTS, USER files), registered skills (SKILL.md), MCP server configs, cron jobs, org hierarchy, and long-term memory stores. Produces a structured inventory of everything in your OpenClaw setup.

**Phase 2 — Plan**
Generates a migration plan and displays it for your review before any files are written. You can inspect what will be created, renamed, or skipped, and abort if something looks wrong.

**Phase 3 — Execute**
Creates agent definition files under `~/.claude/agents/`, merges or installs shared skills, writes per-agent `mcpServers` frontmatter, converts cron jobs to CC scheduled triggers, and seeds agent memory from your existing MEMORY.md files.

## What Gets Migrated

| OpenClaw Feature | CC Equivalent |
|---|---|
| Agent definitions (IDENTITY, SOUL, AGENTS, USER) | `~/.claude/agents/{name}.md` with persistent memory |
| Skills (SKILL.md) | Merged into agent body or `~/.claude/skills/` |
| MCP servers | Per-agent `mcpServers` frontmatter |
| Cron jobs | CC scheduled triggers |
| Long-term memory (MEMORY.md) | `~/.claude/agent-memory/{name}/` |
| Org hierarchy and routing | Lead agent routing instructions |
| Inter-agent communication | Agent Teams teammate messaging |

## What Does Not Migrate

- **Per-agent Telegram/WhatsApp bots** — CC uses a single channel; consolidate to one bot manually
- **Vector/semantic memory search** — CC uses file-based memory; semantic search is not available
- **Daily notes** — Ephemeral by design; not carried over
- **Always-on daemon mode** — Use a persistent tmux session instead

## After Migration

List your migrated agents:

```
/agents
```

Start using them:

```
Create an agent team with [agent-name, agent-name] to [task description]
```

Configure any MCP server credentials manually in your agent frontmatter or `~/.claude/settings.json` — credentials are not copied during migration.

## Uninstall

```bash
claude plugins uninstall claw-to-claude
```

Note: generated agent files at `~/.claude/agents/` are not removed automatically.

## License

MIT
