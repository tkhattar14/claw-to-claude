# From OpenClaw to Claude Code Agent Teams: A Migration Guide

I ran a 13-agent AI team on OpenClaw for months. It managed my startup's operations — finance, marketing, engineering, customer support, daily standups, even a personal coaching agent. Each agent had its own Telegram bot, persistent memory, scheduled tasks, and MCP integrations.

Then Anthropic cut off Claude access through OpenClaw's subscription model. Overnight, a significant chunk of OpenClaw users lost their primary LLM provider.

This post documents how I evaluated Claude Code Agent Teams as a replacement, what maps cleanly, what doesn't, and the open-source migration tool I built to make the switch.

---

## What OpenClaw Actually Does

OpenClaw is a self-hosted AI assistant platform. You run a Gateway daemon on your machine. It connects to messaging channels (Telegram, WhatsApp, Slack, Discord) and routes messages to isolated AI agents.

Each agent is a fully scoped brain:
- Its own workspace with personality files (`SOUL.md`, `AGENTS.md`, `IDENTITY.md`)
- Its own session store with persistent conversation history
- Its own memory (long-term `MEMORY.md` + daily notes with vector search)
- Its own tools, MCP servers, and scheduled tasks

Agents can message each other via `sessions_send`. A CTO agent can spawn backend and frontend engineers. A Chief of Staff agent runs eight daily cron phases — morning prep at 7:30 AM through night wrap at 9 PM.

It works well. The problem is it needs a strong LLM behind it, and the subscription route to Claude is gone.

## What Claude Code Agent Teams Offers

Anthropic launched Agent Teams as an experimental feature in Claude Code v2.1.32. The core idea: multiple Claude Code instances working as a coordinated team.

One session acts as the **lead**. It spawns **teammates** — each a fully independent Claude Code instance with its own context window. Teammates communicate directly with each other through a shared mailbox, coordinate via a shared task list, and self-assign work.

This is architecturally closer to OpenClaw than I expected.

| Concept | OpenClaw | CC Agent Teams |
|---------|----------|----------------|
| Agent definition | Workspace files (SOUL.md, AGENTS.md) | Subagent `.md` files in `~/.claude/agents/` |
| Agent isolation | Separate workspace + session store | Separate context window per teammate |
| Inter-agent messaging | `sessions_send` | Direct teammate messaging |
| Task coordination | Manual or via cron | Shared task list with dependency tracking |
| Persistent memory | SQLite with vector search | `memory: user` field in agent frontmatter |
| Scheduled tasks | Built-in cron system | CC scheduled triggers |
| MCP integrations | Per-agent MCP config | Per-agent `mcpServers` in frontmatter |

## Five Things I Learned During the Migration

### 1. Subagent definitions serve double duty

In Claude Code, you define agents as markdown files with YAML frontmatter at `~/.claude/agents/`. These definitions can be used two ways:

- As **subagents**: one-shot workers that do a task and report back. No peer communication.
- As **Agent Team teammates**: independent instances that talk to each other and self-coordinate.

The same `.md` file works for both. You write it once. How it behaves depends on whether you spawn it as a subagent or as a teammate.

### 2. Plugin agents can't use MCP servers

This one cost me time. Claude Code plugins can include agent definitions, but plugin-sourced agents silently ignore `mcpServers`, `hooks`, and `permissionMode` in their frontmatter. Security restriction.

If your agents need MCP integrations (database tools, payment APIs, analytics), the definitions must live at user level (`~/.claude/agents/`) or project level (`.claude/agents/`). Not in a plugin.

This changed our entire migration architecture. The migration tool generates user-level agent files, not a plugin.

### 3. The `memory: user` field is the persistence answer

OpenClaw agents have rich persistent memory — SQLite databases with vector search, daily notes, long-term `MEMORY.md` files loaded every session.

Claude Code's equivalent is simpler but functional: set `memory: user` in the agent frontmatter and each agent gets a persistent memory directory at `~/.claude/agent-memory/{name}/`. It survives across sessions. Not vector search, but it carries context forward.

### 4. Not everything should be a teammate

My OpenClaw setup runs 13 agents simultaneously. Translating that directly to Agent Teams would mean 13 concurrent Claude instances burning tokens — most of them idle.

The right pattern: the lead agent knows about all 13 agents but spawns only the ones needed for the current task. "Add a coupon feature" spawns the CTO, backend engineer, and frontend engineer as teammates. "Check this month's P&L" spawns only the finance agent. The lead routes, the teammates execute.

### 5. Per-agent messaging channels don't translate

OpenClaw lets you assign a separate Telegram bot to each agent. My marketing agent has its own bot. My finance agent has its own. I message them directly from my phone.

Claude Code has a single Telegram channel. All messages go to the lead session. The lead routes internally. You lose the ability to DM a specific agent from your phone.

This is the biggest workflow gap. Everything else has a reasonable mapping.

## What Migrates and What Doesn't

**Migrates cleanly:**
- Agent identities, personalities, and responsibilities
- Org hierarchy and delegation chains
- Skills (merged into agent body or shared CC skill files)
- MCP server configurations
- Cron/scheduled tasks
- Long-term memory content
- Inter-agent communication patterns

**Doesn't migrate:**
- Per-agent Telegram/WhatsApp bots (CC has one channel)
- Vector/semantic memory search (CC uses file-based grep)
- Always-on daemon (use a persistent tmux session instead)
- Daily notes (ephemeral by design, not worth migrating)

## The Migration Tool

I built [claw-to-claude](https://github.com/tkhattar14/claw-to-claude), an open-source Claude Code plugin that automates the migration. Three phases:

**Analyze**: Reads your OpenClaw config and all workspace files. Maps agents, skills, MCP servers, cron jobs, and org hierarchy.

**Plan**: Generates a migration plan document — agent mapping table, skill consolidation decisions, MCP config, scheduled tasks, and an honest list of what won't migrate. You review and edit before anything is written.

**Execute**: Generates agent definitions at `~/.claude/agents/`, creates shared skill files, seeds persistent memory from your existing `MEMORY.md` files, and prints setup instructions.

Install:
```
claude plugins install github:tkhattar14/claw-to-claude
```

Run:
```
/claw-to-claude:migrate --path ~/openclaw-snapshot
```

Or directly from your OpenClaw host:
```
/claw-to-claude:migrate --ssh user@your-openclaw-host
```

## Is It a Full Replacement?

No. OpenClaw is purpose-built for always-on, multi-channel AI assistants. Agent Teams is built for collaborative development work within Claude Code sessions.

If your workflow is "message my finance agent from WhatsApp at 2 PM and get a P&L update" — that stays on OpenClaw (with a different LLM provider).

If your workflow is "break down this feature across backend and frontend, have them coordinate, and ship it" — Agent Teams handles this well and the migration is straightforward.

For me, the split is: operations stay on OpenClaw, development moves to Claude Code. The agents are defined in the same way. The memory carries over. The routing logic adapts. It's not a replacement — it's a complement.

---

*The migration tool is at [github.com/tkhattar14/claw-to-claude](https://github.com/tkhattar14/claw-to-claude). MIT licensed. Issues and PRs welcome.*
