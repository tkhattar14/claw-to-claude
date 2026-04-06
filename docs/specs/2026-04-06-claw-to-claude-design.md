# claw-to-claude: OpenClaw to CC Agent Teams Migration Skill

**Date:** 2026-04-06
**Status:** Draft

## Overview

`claw-to-claude` is a Claude Code plugin that migrates an OpenClaw multi-agent setup to Claude Code Agent Teams. It reads OpenClaw configuration and workspace files, produces a migration plan for user review, then generates CC agent definitions with persistent memory, MCP configs, and scheduled tasks.

**Input:** OpenClaw snapshot directory (local) or SSH access to OpenClaw host
**Output:** User-level CC agent definitions at `~/.claude/agents/`, shared skills, and post-migration instructions

## Target User

OpenClaw power users who want to migrate their complete multi-agent setup to Claude Code Agent Teams. The skill assumes a working OpenClaw installation with configured agents, workspaces, skills, and optionally MCP servers, cron jobs, and messaging channels.

## Three-Phase Process

### Phase 1: Analyze

Reads the OpenClaw setup and extracts a structured inventory.

#### Input Options

- `--path /path/to/openclaw-snapshot` — reads from a local directory (e.g. after SCP)
- `--ssh user@host` — reads live from the OpenClaw host via SSH

#### What It Reads

| Source | OpenClaw Location | What It Extracts |
|---|---|---|
| Agent list | `openclaw.json` → `agents.list[]` | IDs, names, workspaces, models, default agent |
| Agent identity | `IDENTITY.md` per workspace | Name, emoji, role title |
| Agent personality | `SOUL.md` per workspace | Tone, hard rules, behavioral constraints |
| Agent instructions | `AGENTS.md` per workspace | Responsibilities, standing orders, delegation rules |
| User context | `USER.md` per workspace | Who the human is, preferences |
| Bootstrap | `BOOTSTRAP.md` per workspace | Initial setup instructions (if present) |
| Tools config | `openclaw.json` → `agents.list[].tools` | Per-agent allow/deny lists |
| Routing/bindings | `openclaw.json` → `bindings[]` | Channel → agent mapping |
| Inter-agent access | `openclaw.json` → `tools.agentToAgent` | Who can spawn/message whom |
| Skills | `workspace/skills/*/SKILL.md` | Skill name, content, which agents reference them |
| MCP servers | `openclaw.json` → MCP/mcporter config | Server names, types, tool counts |
| Cron jobs | `openclaw.json` → cron config or `openclaw cron list` output | Schedule, target agent, prompt |
| Long-term memory | `MEMORY.md` per workspace | Durable facts, decisions, preferences |
| Daily notes | `memory/YYYY-MM-DD.md` per workspace | Recent context (not migrated, noted in plan) |
| Org hierarchy | Derived from AGENTS.md cross-references | Reporting chains, delegation permissions |

### Phase 2: Plan

Generates a migration plan document for user review. The user can edit the plan before execution.

#### Plan Contents

**1. Agent Mapping Table**

For each OpenClaw agent:
- Agent name/ID and role
- CC agent file path: `~/.claude/agents/{name}.md`
- Model mapping (OpenClaw model → CC model equivalent)
- Memory scope: `user` (persistent cross-session)
- Tools allow/deny list
- MCP servers required
- Skills being merged into agent body vs referenced as shared

**2. Org Hierarchy & Lead Definition**

- Which agent becomes the team lead (typically `main` or the default agent)
- Lead's routing instructions: when to spawn which teammates
- Delegation chains (e.g. CTO spawns Backend/Frontend/QA)
- Guidance on when to use Agent Teams (collaborative cross-cutting work) vs single teammate spawns (focused tasks)

**3. Skill Consolidation Map**

- Skills used by only one agent → merged into that agent's `.md` body
- Skills used by multiple agents → kept as separate CC skill files at `~/.claude/skills/{name}.md`, referenced via `skills` frontmatter

**4. MCP Server Config**

- Which MCP servers need configuring in CC
- Placement: user `settings.json` or per-agent `mcpServers` frontmatter
- Credentials/tokens the user needs to provide (not auto-configured)

**5. Scheduled Tasks**

- Each OpenClaw cron job → equivalent CC scheduled trigger
- Schedule expression, target agent, prompt content

**6. What Won't Migrate**

- Per-agent messaging channels (multiple Telegram bots → single CC Telegram channel)
- WhatsApp integrations (no CC equivalent)
- Vector/semantic memory search (CC uses file-based grep)
- Any setup-specific gaps detected during analysis

**7. Post-Migration Checklist**

Manual steps after execution:
- Configure MCP server credentials
- Enable `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` in settings
- Set up Telegram channel (if desired)
- Start tmux persistent session (for always-on behavior)
- Test first team spawn

### Phase 3: Execute

After user approves the plan, generates all artifacts.

#### 1. Agent Definition Files

For each agent, creates `~/.claude/agents/{name}.md`:

```yaml
---
name: {name}
description: {one-line role description}
model: {mapped CC model}
tools: {allowed tools}
disallowedTools: {denied tools}
mcpServers: {per-agent MCP servers, if any}
memory: user
skills: {list of shared skill references}
---
```

Body content is merged from OpenClaw workspace files in this order:
1. **Identity** — who you are (from IDENTITY.md)
2. **Personality & rules** — tone, constraints (from SOUL.md)
3. **Responsibilities & standing orders** — what you do (from AGENTS.md)
4. **User context** — who you serve (from USER.md)
5. **Embedded skills** — skills used only by this agent (from relevant SKILL.md files)

#### 2. Lead Agent Definition

The lead agent (typically the OpenClaw default agent) gets additional instructions:
- Full org hierarchy with agent descriptions
- Routing logic: "when the user asks about X, spawn Y as teammate"
- Cross-cutting work patterns: which agents to spawn together
- Guidance on teammate vs single-agent decisions

#### 3. Shared Skills

Skills used by multiple agents are written to `~/.claude/skills/{skill-name}.md` and referenced via the `skills` frontmatter field in each agent definition that needs them.

#### 4. Memory Seeding

Copies each agent's `MEMORY.md` content into `~/.claude/agent-memory/{name}/` so agents start with their existing long-term knowledge. Daily notes (`memory/YYYY-MM-DD.md`) are not copied — they are ephemeral by design.

#### 5. Post-Migration Output

Prints to the user:
- Summary: X agents generated, Y shared skills, Z MCP configs needed
- Commands to set up scheduled tasks
- Telegram setup instructions
- How to start: example prompt to create first agent team
- Path to migration plan document for reference

## What the Skill Does NOT Do

- Does not configure MCP server credentials (user must provide tokens/keys)
- Does not auto-enable Agent Teams experimental flag (instructs user how)
- Does not set up Telegram bot (instructs user how)
- Does not create tmux sessions (instructs user how)
- Does not modify `~/.claude/settings.json` without explicit user confirmation
- Does not migrate daily notes (ephemeral by nature)

## Model Mapping

The skill maps OpenClaw model identifiers to CC equivalents:

| OpenClaw Model | CC Equivalent |
|---|---|
| `anthropic/claude-opus-4-6` | `opus` |
| `anthropic/claude-sonnet-4-6` | `sonnet` |
| `anthropic/claude-haiku-4-5` | `haiku` |
| `openai/*` or other providers | User prompted to choose CC model |

When the OpenClaw setup uses non-Anthropic models (e.g. `openai-codex/gpt-5.4`), the plan flags this and asks the user to select a CC model for each agent.

## Repo Structure

```
claw-to-claude/
├── README.md
├── LICENSE
├── plugin.json
├── skills/
│   └── claw-to-claude/
│       └── SKILL.md
├── commands/
│   └── migrate.md
└── examples/
    ├── sample-openclaw-config/
    └── sample-output/
```

Installed as a CC plugin via `claude plugins install /path/to/claw-to-claude` or from GitHub.

Invoked via `/migrate --path ~/openclaw-snapshot` or `/migrate --ssh user@host`.

## Error Handling

- **Missing files:** If expected OpenClaw files are missing (e.g. no SOUL.md for an agent), the skill notes the gap in the plan and generates the agent without that section.
- **Unparseable config:** If `openclaw.json` can't be parsed, the skill reports the error and exits.
- **SSH failures:** If SSH connection fails, the skill suggests the local snapshot approach.
- **Existing CC agents:** If `~/.claude/agents/{name}.md` already exists, the skill asks whether to overwrite, skip, or merge.
- **Unknown MCP servers:** MCP servers with no obvious CC equivalent are listed in "What Won't Migrate" with the original config for reference.

## Success Criteria

After migration, a user should be able to:
1. See all their agents via `/agents` in Claude Code
2. Say "Create an agent team with [agent names] to [task]" and have teammates spawn with correct roles, tools, and MCP access
3. Have each agent retain long-term memory across sessions via `memory: user`
4. Reference the migration plan document for what mapped where and what needs manual setup
