# claw-to-claude Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that migrates an OpenClaw multi-agent setup to CC Agent Teams by reading OpenClaw config/workspace files and generating user-level agent definitions.

**Architecture:** A CC plugin containing a single command (`/claw-to-claude:migrate`). The command's markdown file contains all instructions for Claude to execute a three-phase migration: Analyze → Plan → Execute. No code — the plugin is pure prompt engineering in markdown.

**Tech Stack:** Claude Code plugin system (plugin.json + commands/*.md), Bash for file I/O, Markdown for all output.

---

## File Structure

```
claw-to-claude/
├── .claude-plugin/
│   └── plugin.json                    # Plugin manifest
├── commands/
│   └── migrate.md                     # The /claw-to-claude:migrate command (core logic)
├── README.md                          # Installation, usage, what it does
├── LICENSE                            # MIT
└── examples/
    └── sample-output/
        ├── lead.md                    # Example lead agent definition
        ├── backend-engineer.md        # Example teammate agent definition
        └── migration-plan.md          # Example migration plan output
```

---

### Task 1: Create plugin manifest

**Files:**
- Create: `claw-to-claude/.claude-plugin/plugin.json`

- [ ] **Step 1: Create the plugin.json**

```json
{
  "name": "claw-to-claude",
  "description": "Migrate your OpenClaw multi-agent setup to Claude Code Agent Teams. Reads OpenClaw config and workspace files, generates CC agent definitions with persistent memory, MCP configs, and scheduled tasks.",
  "version": "0.1.0",
  "author": {
    "name": "OnceUponMe"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/onceuponme/claw-to-claude"
  },
  "license": "MIT"
}
```

- [ ] **Step 2: Verify plugin structure**

Run: `ls -la claw-to-claude/.claude-plugin/`
Expected: `plugin.json` exists

- [ ] **Step 3: Commit**

```bash
cd ~/Documents/matrix/claw-to-claude
git add .claude-plugin/plugin.json
git commit -m "feat: add plugin manifest"
```

---

### Task 2: Create the migrate command

This is the core of the plugin — a single command file that contains all instructions for Claude to execute the three-phase migration.

**Files:**
- Create: `claw-to-claude/commands/migrate.md`

- [ ] **Step 1: Create the command file with frontmatter**

The command file needs:
- Frontmatter with description for `/claw-to-claude:migrate`
- `$ARGUMENTS` placeholder to accept `--path` or `--ssh` input
- Phase 1: Analyze instructions (how to read OpenClaw files)
- Phase 2: Plan instructions (how to generate migration plan)
- Phase 3: Execute instructions (how to generate CC agent definitions)

The command content should include:

**Frontmatter:**
```yaml
---
description: "Migrate an OpenClaw setup to Claude Code Agent Teams. Usage: /claw-to-claude:migrate --path /path/to/snapshot OR /claw-to-claude:migrate --ssh user@host"
---
```

**Body sections:**

**A. Argument Parsing**
- Parse `$ARGUMENTS` for `--path <dir>` or `--ssh user@host`
- If `--ssh`, create a local snapshot first by running SCP/SSH commands to pull config, workspace files, skills, memory, and status outputs into a temp directory
- If `--path`, validate the directory exists and contains expected OpenClaw files (`openclaw.json` or equivalent)
- If neither provided, ask the user which input method they prefer

**B. Phase 1: Analyze**

Instructions for Claude to:
1. Read `openclaw.json` (or `openclaw.json5`) and parse the agent list from `agents.list[]`
2. For each agent, read workspace files:
   - `~/.openclaw/workspace-{agentId}/IDENTITY.md` (or `workspace/` for default)
   - `~/.openclaw/workspace-{agentId}/SOUL.md`
   - `~/.openclaw/workspace-{agentId}/AGENTS.md`
   - `~/.openclaw/workspace-{agentId}/USER.md`
   - `~/.openclaw/workspace-{agentId}/MEMORY.md`
3. Read skills from `workspace/skills/*/SKILL.md` and per-agent workspace skills
4. Extract from config:
   - `bindings[]` — channel routing rules
   - `agents.list[].tools` — per-agent tool allow/deny
   - `tools.agentToAgent` — inter-agent access
   - MCP/mcporter server configurations
   - Cron job definitions
5. Derive org hierarchy from AGENTS.md cross-references (who reports to whom, who delegates to whom)
6. Identify the default/lead agent (`agents.list[].default: true` or first in list)
7. Build a skill usage map: which skills are referenced by which agents (scan AGENTS.md files for skill mentions, check skill directory locations)
8. Present a summary to the user:
   - "Found X agents, Y skills, Z MCP servers, W cron jobs"
   - List each agent with name and role
   - Ask: "Proceed to migration plan?"

**C. Phase 2: Plan**

Instructions for Claude to generate a migration plan document:

1. **Agent Mapping Table** — for each agent:
   - OpenClaw agent ID → CC agent file name (`~/.claude/agents/{name}.md`)
   - Role (from IDENTITY.md)
   - Model mapping:
     - `anthropic/claude-opus-4-6` → `opus`
     - `anthropic/claude-sonnet-4-6` → `sonnet`
     - `anthropic/claude-haiku-4-5` → `haiku`
     - Any `openai/*` or other provider → flag for user to choose, default to `inherit`
   - Memory: `user` (all agents)
   - Tools: map OpenClaw allow/deny to CC `tools`/`disallowedTools` frontmatter
   - MCP servers: list which servers this agent needs

2. **Org Hierarchy & Lead** — identify:
   - Which agent is the lead (the default agent)
   - The lead's routing table: "when user asks about [domain], spawn [agent] as teammate"
   - Delegation chains (e.g., CTO agent spawns engineer agents)
   - Which agents work together for cross-cutting tasks

3. **Skill Consolidation** — for each skill:
   - If used by 1 agent → merge into that agent's body
   - If used by 2+ agents → create `~/.claude/skills/{name}.md`, reference via `skills` frontmatter
   - List each skill and its disposition

4. **MCP Server Config** — for each MCP server:
   - Server name, type, endpoint
   - Whether it goes in user `settings.json` or per-agent `mcpServers` frontmatter
   - Note: credentials must be configured separately by user

5. **Scheduled Tasks** — for each cron job:
   - OpenClaw schedule → CC schedule expression
   - Target agent
   - Prompt content

6. **What Won't Migrate** — detect and list:
   - Per-agent Telegram bots (CC has single channel)
   - WhatsApp integrations
   - Vector/semantic memory search
   - Any other gaps

7. **Post-Migration Checklist** — manual steps after execution

Write the plan to `~/claw-to-claude-migration-plan.md` (or user-specified path).

Ask user: "Review the migration plan above. Edit if needed, then say 'execute' to proceed."

**D. Phase 3: Execute**

Instructions for Claude to generate all artifacts:

1. **Create agent definition files** — for each agent in the plan:
   - Create `~/.claude/agents/{name}.md`
   - Frontmatter: `name`, `description`, `model`, `tools`, `disallowedTools`, `mcpServers`, `memory: user`, `skills`
   - Body: merge IDENTITY + SOUL + AGENTS + USER + embedded skills in that order
   - Use `## Identity`, `## Personality`, `## Responsibilities`, `## User Context`, `## Skills` as section headers in the body

2. **Create lead agent definition** — the default/main agent gets additional body content:
   - `## Team Roster` — list all agents with one-line descriptions
   - `## Routing Guide` — when to spawn which agents as teammates
   - `## Cross-Cutting Patterns` — which agents to spawn together for multi-domain tasks
   - `## When to Use Teams vs Single Agent` — guidance on Agent Teams (collaborative) vs single spawns (focused tasks)

3. **Create shared skill files** — for multi-agent skills:
   - Write to `~/.claude/skills/{skill-name}.md`
   - Include appropriate frontmatter

4. **Seed memory** — for each agent with MEMORY.md content:
   - Create `~/.claude/agent-memory/{name}/MEMORY.md`
   - Copy the content from OpenClaw's MEMORY.md

5. **Print post-migration summary**:
   - Count of agents, skills, MCP configs generated
   - Commands to enable Agent Teams: `Add "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" to env in settings.json`
   - Commands to set up scheduled tasks (one per cron job)
   - Example first command: "Try: Create an agent team with [2-3 agent names] to [sample task based on their roles]"
   - Remind about MCP credential setup
   - Path to migration plan document

**E. Error Handling**

- If a workspace file is missing, skip that section and note it in the plan
- If `openclaw.json` can't be found or parsed, report error and exit
- If SSH connection fails, suggest `--path` approach with SCP instructions
- If `~/.claude/agents/{name}.md` already exists, ask: overwrite / skip / merge
- If an MCP server type is unrecognized, list it in "What Won't Migrate" with raw config

- [ ] **Step 2: Write the complete migrate.md file**

Write the full command file combining all sections above into a single markdown document with YAML frontmatter.

- [ ] **Step 3: Verify the command file**

Run: `wc -l claw-to-claude/commands/migrate.md`
Expected: File exists, substantial content (200+ lines)

- [ ] **Step 4: Commit**

```bash
cd ~/Documents/matrix/claw-to-claude
git add commands/migrate.md
git commit -m "feat: add migrate command with three-phase migration logic"
```

---

### Task 3: Create example output files

Example files help users understand what the migration produces.

**Files:**
- Create: `claw-to-claude/examples/sample-output/lead.md`
- Create: `claw-to-claude/examples/sample-output/backend-engineer.md`
- Create: `claw-to-claude/examples/sample-output/migration-plan.md`

- [ ] **Step 1: Create example lead agent definition**

Create `examples/sample-output/lead.md` showing what a migrated lead agent looks like — frontmatter with model/tools/memory, body with identity + routing guide + team roster.

- [ ] **Step 2: Create example teammate agent definition**

Create `examples/sample-output/backend-engineer.md` showing a migrated teammate — frontmatter with tools/mcpServers/memory, body with identity + personality + responsibilities + embedded skills.

- [ ] **Step 3: Create example migration plan**

Create `examples/sample-output/migration-plan.md` showing what Phase 2 produces — agent mapping table, org hierarchy, skill consolidation, MCP config, scheduled tasks, gaps, checklist.

- [ ] **Step 4: Commit**

```bash
cd ~/Documents/matrix/claw-to-claude
git add examples/
git commit -m "docs: add example migration output files"
```

---

### Task 4: Create README

**Files:**
- Create: `claw-to-claude/README.md`

- [ ] **Step 1: Write README with sections:**

1. **What this is** — one-paragraph description
2. **Prerequisites** — OpenClaw setup, Claude Code with Agent Teams enabled
3. **Installation** — `claude plugins install github:onceuponme/claw-to-claude`
4. **Usage** — two methods (local snapshot, SSH)
5. **What it does** — three phases explained briefly
6. **What gets migrated** — table of OpenClaw features → CC equivalents
7. **What doesn't migrate** — honest gaps
8. **Example** — show a before (OpenClaw agent list) → after (CC /agents output)
9. **Uninstall** — `claude plugins uninstall claw-to-claude` (generated agents remain)

- [ ] **Step 2: Commit**

```bash
cd ~/Documents/matrix/claw-to-claude
git add README.md
git commit -m "docs: add README with installation and usage"
```

---

### Task 5: Create LICENSE

**Files:**
- Create: `claw-to-claude/LICENSE`

- [ ] **Step 1: Add MIT license**

Standard MIT license with OnceUponMe as copyright holder, year 2026.

- [ ] **Step 2: Commit**

```bash
cd ~/Documents/matrix/claw-to-claude
git add LICENSE
git commit -m "chore: add MIT license"
```

---

### Task 6: Test with real OpenClaw snapshot

**Files:**
- Read: `~/openclaw-snapshot/` (your OpenClaw snapshot directory)

- [ ] **Step 1: Load the plugin locally**

Run: `claude --plugin-dir ~/claw-to-claude`

- [ ] **Step 2: Run the migration command**

Run: `/claw-to-claude:migrate --path ~/openclaw-snapshot`

- [ ] **Step 3: Verify Phase 1 output**

Expected: Claude reads the snapshot, reports "Found N agents, M skills, X MCP servers, K cron jobs"

- [ ] **Step 4: Verify Phase 2 output**

Expected: Migration plan document generated with all agents mapped, org hierarchy identified, skill consolidation decided

- [ ] **Step 5: Verify Phase 3 output**

Expected: Agent `.md` files in `~/.claude/agents/`, shared skills in `~/.claude/skills/`, memory seeded

- [ ] **Step 6: Verify agents appear**

Run `/agents` in Claude Code — all migrated agents should be listed.

- [ ] **Step 7: Test Agent Team spawn**

Run: "Create an agent team with backend-engineer, frontend-engineer, and data-analyst to review the architecture"
Expected: Three teammates spawn with correct roles and tools.

- [ ] **Step 8: Fix any issues found and commit**

```bash
cd ~/claw-to-claude
git add -A
git commit -m "fix: address issues found during real-world testing"
```

---

### Task 7: Final review and push

- [ ] **Step 1: Review all generated files**

Check plugin.json, commands/migrate.md, examples/*, README.md, LICENSE for completeness.

- [ ] **Step 2: Create GitHub repo and push**

```bash
cd ~/Documents/matrix/claw-to-claude
gh repo create onceuponme/claw-to-claude --public --description "Migrate OpenClaw multi-agent setups to Claude Code Agent Teams" --source . --push
```
