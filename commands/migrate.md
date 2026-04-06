---
description: "Migrate an OpenClaw multi-agent setup to Claude Code Agent Teams. Usage: /claw-to-claude:migrate --path /path/to/snapshot OR /claw-to-claude:migrate --ssh user@host"
argument-hint: "--path /path/to/openclaw-snapshot OR --ssh user@host"
---

# OpenClaw to Claude Code Agent Teams Migration

You are executing a three-phase migration that converts an OpenClaw multi-agent setup into Claude Code Agent Teams. Read the OpenClaw configuration and workspace files, generate a migration plan for user review, then produce CC agent definitions, shared skills, and seeded memory.

**User-provided arguments:** $ARGUMENTS

---

## Step 0: Parse Arguments

Parse `$ARGUMENTS` to determine the input source:

1. **If `--path <directory>` is present:**
   - Set `SNAPSHOT_DIR` to the provided path
   - Verify the directory exists: `ls "$SNAPSHOT_DIR"`
   - If it does not exist, tell the user and stop

2. **If `--ssh <user@host>` is present:**
   - Test SSH connectivity: `ssh -o ConnectTimeout=10 <user@host> "echo ok"`
   - If SSH fails, tell the user:
     > SSH connection failed. You can create a local snapshot instead:
     > ```
     > mkdir -p ~/openclaw-snapshot
     > scp -r user@host:~/.openclaw/config/ ~/openclaw-snapshot/config/
     > scp -r user@host:~/.openclaw/workspace* ~/openclaw-snapshot/
     > ```
     > Then run: `/claw-to-claude:migrate --path ~/openclaw-snapshot`
   - If SSH succeeds, create a local snapshot:
     ```bash
     SNAPSHOT_DIR=$(mktemp -d)/openclaw-snapshot
     mkdir -p "$SNAPSHOT_DIR"
     scp -r <user@host>:~/.openclaw/config/ "$SNAPSHOT_DIR/config/"
     scp -r <user@host>:~/.openclaw/workspace* "$SNAPSHOT_DIR/"
     ```
   - Tell the user: "Pulled snapshot to $SNAPSHOT_DIR"

3. **If neither is provided:**
   - Ask the user: "Please provide either `--path /path/to/snapshot` (local directory) or `--ssh user@host` (live OpenClaw host). If you have the files locally, use `--path`."
   - Stop and wait for their response

---

## Phase 1: Analyze

**Goal:** Read the OpenClaw snapshot and build a complete inventory of agents, skills, MCP servers, cron jobs, and org hierarchy.

### 1.1 Locate and parse openclaw.json

Search for the OpenClaw config file. It may be at any of these locations within the snapshot:

```
$SNAPSHOT_DIR/openclaw.json
$SNAPSHOT_DIR/openclaw.json5
$SNAPSHOT_DIR/config/openclaw.json
$SNAPSHOT_DIR/config/openclaw.json5
$SNAPSHOT_DIR/.openclaw/config/openclaw.json
```

Use Bash with `find "$SNAPSHOT_DIR" -name "openclaw.json*" -type f` if none of the above exist.

**If no config file is found:** Report the error and stop:
> Could not find openclaw.json in the snapshot directory. Expected it at `$SNAPSHOT_DIR/config/openclaw.json` or `$SNAPSHOT_DIR/openclaw.json`. Please verify your snapshot contains the OpenClaw configuration.

**If found:** Read the file. Note that OpenClaw uses JSON5 format (allows comments and trailing commas). If the file has comments (`//` or `/* */`), strip them before parsing. Use Bash to strip comments if needed:
```bash
sed 's|//.*$||g; s|/\*.*\*/||g' openclaw.json | python3 -c "import sys,json; print(json.dumps(json.loads(sys.stdin.read())))"
```
If that fails, try reading it raw and parsing manually from the text content.

Extract these sections from the parsed config:

- **`agents.list[]`** -- array of agent objects. For each agent, extract:
  - `id` (string identifier)
  - `name` (display name)
  - `workspace` (workspace directory path, if specified)
  - `agentDir` (agent directory path, if specified)
  - `model` (model identifier, e.g. `anthropic/claude-sonnet-4-6`)
  - `tools.allow` (array of allowed tool names)
  - `tools.deny` (array of denied tool names)
  - `default` (boolean -- true means this is the lead/default agent)

- **`bindings[]`** -- routing rules mapping channels/patterns to agents

- **`tools.agentToAgent`** -- inter-agent communication configuration (who can spawn/message whom)

- **MCP/mcporter configuration** -- any MCP server definitions (server names, types, endpoints, tool lists)

- **Cron jobs** -- scheduled task definitions (schedule expression, target agent, prompt)

### 1.2 Read workspace files for each agent

For each agent in `agents.list[]`, locate and read its workspace files. The workspace layout depends on the snapshot structure:

**Standard layout** (mirroring OpenClaw's filesystem):
```
$SNAPSHOT_DIR/workspace/              # default agent's workspace
$SNAPSHOT_DIR/workspace-{agentId}/    # other agents' workspaces
```

**Flat snapshot layout** (filenames encode the workspace):
```
$SNAPSHOT_DIR/workspace_IDENTITY.md           # default agent
$SNAPSHOT_DIR/workspace-{agentId}_IDENTITY.md # other agents
```

**Nested under .openclaw:**
```
$SNAPSHOT_DIR/.openclaw/workspace/
$SNAPSHOT_DIR/.openclaw/workspace-{agentId}/
```

For each agent, try to read these files (skip any that do not exist):

| File | Content | Required |
|------|---------|----------|
| `IDENTITY.md` | Name, emoji, role title | Recommended |
| `SOUL.md` | Personality, tone, hard behavioral rules | Recommended |
| `AGENTS.md` | Responsibilities, standing orders, delegation rules, skill references | Recommended |
| `USER.md` | Information about the human user | Optional |
| `MEMORY.md` | Long-term memory (facts, decisions, preferences) | Optional |
| `BOOTSTRAP.md` | Initial setup instructions | Optional |
| `TOOLS.md` | Tool usage instructions | Optional |

For missing files, note the gap but continue. An agent with no workspace files at all should still be included in the inventory with a warning.

### 1.3 Read skills

Skills live in workspace skill directories:

```
$SNAPSHOT_DIR/workspace/skills/*/SKILL.md          # default agent's skills (or shared)
$SNAPSHOT_DIR/workspace-{agentId}/skills/*/SKILL.md # per-agent skills
```

Also check for a shared skills directory:
```
$SNAPSHOT_DIR/skills/*/SKILL.md
$SNAPSHOT_DIR/.openclaw/skills/*/SKILL.md
```

For each skill found, record:
- Skill name (from directory name)
- Skill content (from SKILL.md)
- Which workspace(s) it appears in

### 1.4 Derive org hierarchy

Scan each agent's `AGENTS.md` for cross-references to other agents. Look for:
- Mentions of other agent names or IDs
- Phrases like "reports to", "delegates to", "spawns", "coordinates with", "escalates to"
- Delegation rules ("when X happens, hand off to Y")
- Skill references (which skills does this agent mention by name?)

Build a hierarchy map:
- Which agent is the **lead** (has `default: true` in config, or is first in list)
- Which agents **delegate to** which others
- Which agents **collaborate** on shared domains

### 1.5 Build skill usage map

For each skill found in step 1.3, determine which agents reference it:
- Scan each agent's `AGENTS.md` for mentions of the skill name
- Check if the skill exists in the agent's own workspace skills directory
- Check if the skill exists in the shared skills directory

Classify each skill:
- **Single-agent skill:** Referenced by exactly 1 agent -- will be merged into that agent's body
- **Multi-agent skill:** Referenced by 2+ agents -- will become a shared CC skill file

### 1.6 Present summary

Display the analysis results to the user in this format:

```
## OpenClaw Analysis Complete

Found **X agents**, **Y skills**, **Z MCP servers**, **W cron jobs**

### Agents
| # | ID | Name | Role | Model | Default |
|---|-----|------|------|-------|---------|
| 1 | main | [name] | [from IDENTITY.md] | [model] | Yes |
| 2 | ... | ... | ... | ... | No |

### Org Hierarchy
- Lead: [agent name]
- [agent] delegates to: [agents]
- [agent] collaborates with: [agents]

### Skills
- X single-agent skills (will merge into agent bodies)
- Y multi-agent skills (will become shared CC skills)

### MCP Servers
- [list each server name and type]

### Cron Jobs
- [list each with schedule and target agent]

### Gaps Detected
- [any missing workspace files, unreadable configs, etc.]
```

Then ask: **"Proceed to migration plan? (yes/no)"**

Wait for user confirmation before continuing to Phase 2.

---

## Phase 2: Plan

**Goal:** Generate a migration plan document for user review. The user can edit it before execution.

Write the plan to `~/claw-to-claude-migration-plan.md` using the Write tool. The plan document should contain these sections:

### Plan Document Structure

```markdown
# OpenClaw to Claude Code Agent Teams Migration Plan

Generated: [current date]
Source: [snapshot path]

---

## 1. Agent Mapping Table

| OpenClaw ID | Role | CC File | Model | Memory | Tools | MCP Servers |
|-------------|------|---------|-------|--------|-------|-------------|
| [id] | [role from IDENTITY] | ~/.claude/agents/[name].md | [mapped model] | user | [tools] | [servers] |

### Model Mapping Applied
[For each agent, show the mapping:]
- [agent]: [openclaw model] -> [cc model]
[Flag any non-Anthropic models that need user decision]

## 2. Org Hierarchy & Lead

**Lead agent:** [name] (from OpenClaw default agent)

### Routing Table
| User asks about... | Spawn agent... | Reason |
|---------------------|----------------|--------|
| [domain] | [agent name] | [from AGENTS.md delegation rules] |

### Delegation Chains
- [lead] -> [agent] -> [sub-agent] (if any)

### Cross-Cutting Patterns
- [task type] -> spawn [agent A] + [agent B] together

## 3. Skill Consolidation

### Single-Agent Skills (merge into agent body)
| Skill | Agent | Action |
|-------|-------|--------|
| [name] | [agent] | Merge into ## Skills section |

### Multi-Agent Skills (create shared CC skills)
| Skill | Used By | CC Path |
|-------|---------|---------|
| [name] | [agent1, agent2] | ~/.claude/skills/[name].md |

## 4. MCP Server Config

| Server | Type | Placement | Credentials Needed |
|--------|------|-----------|-------------------|
| [name] | [type] | settings.json / per-agent | [what user needs to provide] |

## 5. Scheduled Tasks

| OpenClaw Cron | Schedule | Agent | Prompt | CC Equivalent |
|---------------|----------|-------|--------|---------------|
| [name] | [expression] | [agent] | [prompt] | CC schedule with same expression |

## 6. What Won't Migrate

The following OpenClaw features have no direct CC equivalent:

- **Per-agent Telegram bots** -- OpenClaw supports multiple Telegram bots (one per agent). CC has a single Telegram channel. All agents will share one channel.
- **WhatsApp integrations** -- No CC equivalent. WhatsApp routing will need a separate solution.
- **Vector/semantic memory search** -- OpenClaw's vector search becomes file-based grep in CC. MEMORY.md content is preserved but search behavior changes.
- **Daily notes (memory/YYYY-MM-DD.md)** -- Ephemeral by design, not migrated.
[Add any other detected gaps]

## 7. Post-Migration Checklist

After execution, complete these manual steps:

- [ ] Enable Agent Teams: add `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"` to `env` in `~/.claude/settings.json`
- [ ] Configure MCP server credentials (see Section 4 for details)
- [ ] Set up Telegram channel (if desired): configure bot token in CC settings
- [ ] Start tmux persistent session for always-on behavior
- [ ] Test first team spawn: "Create an agent team with [2-3 agent names] to [sample task]"
- [ ] Verify each agent appears in /agents
- [ ] Test individual agent spawns for key workflows
```

### Model Mapping Rules

Apply these mappings when generating the plan:

| OpenClaw Model | CC Model |
|----------------|----------|
| `anthropic/claude-opus-4-6` | `opus` |
| `anthropic/claude-sonnet-4-6` | `sonnet` |
| `anthropic/claude-haiku-4-5` | `haiku` |
| Any `openai/*` or other non-Anthropic | Flag for user, suggest `inherit` |

For non-Anthropic models, add a note in the plan:
> **Action required:** Agent "[name]" uses `[model]` which has no CC equivalent. Defaulting to `inherit` (uses the model of the parent session). Change this in the agent definition if you prefer a specific model.

### After writing the plan

Tell the user:
> Migration plan written to `~/claw-to-claude-migration-plan.md`.
>
> Please review the plan. You can edit it to adjust model mappings, change agent names, modify skill consolidation decisions, or update routing rules.
>
> When ready, say **"execute"** to proceed with generating all artifacts.

Wait for the user to say "execute" before continuing to Phase 3.

---

## Phase 3: Execute

**Goal:** Generate all CC agent definitions, shared skills, and seeded memory based on the approved plan.

### 3.0 Check for existing agents

Before writing any files, check if any target agent files already exist:

```bash
ls ~/.claude/agents/ 2>/dev/null
```

For each agent in the plan, if `~/.claude/agents/{name}.md` already exists, ask the user:
> Agent file `~/.claude/agents/{name}.md` already exists. What should I do?
> - **overwrite** -- Replace with migrated version
> - **skip** -- Keep existing, do not migrate this agent
> - **merge** -- Append migrated content below existing content

Apply the user's choice for each conflict. If there are many conflicts, ask once: "Should I overwrite all, skip all, or ask for each?"

### 3.1 Create agent definition files

Create the directory if it does not exist:
```bash
mkdir -p ~/.claude/agents
```

For **each non-lead agent**, create `~/.claude/agents/{name}.md` with this structure:

```markdown
---
name: {name}
description: {one-line role description derived from IDENTITY.md}
model: {mapped CC model: opus | sonnet | haiku | inherit}
tools: {comma-separated allowed tools from OpenClaw config, e.g. Read, Grep, Glob, Bash, Write, Edit}
disallowedTools: {comma-separated denied tools, omit if empty}
mcpServers:
  {server-name}:
    {inline config or reference, only if this agent specifically needs it}
memory: user
skills: {comma-separated shared skill names from skill consolidation, omit if none}
---

## Identity

{Content from IDENTITY.md -- name, emoji, role title, one-line description}

## Personality & Rules

{Content from SOUL.md -- tone, behavioral constraints, hard rules}

## Responsibilities

{Content from AGENTS.md -- standing orders, delegation rules, what this agent does}
{Remove cross-references to OpenClaw-specific features (channel routing, OpenClaw commands)}
{Adapt delegation references to use CC Agent Teams language: "spawn as teammate" instead of "message agent"}

## User Context

{Content from USER.md -- who the human is, their preferences}

## Skills

{For each single-agent skill that maps to this agent, include a subsection:}

### {Skill Name}

{Content from SKILL.md}
```

**Important adaptation rules when writing body content:**
- Replace OpenClaw-specific terminology: "message agent X" becomes "spawn X as teammate"
- Replace "workspace" references with CC equivalents
- Remove references to OpenClaw commands, channels, or Telegram bot routing
- Keep all personality, rules, responsibilities, and domain knowledge intact
- Preserve the agent's voice and character from SOUL.md faithfully

### 3.2 Create lead agent definition

The lead agent (the one with `default: true` or first in list) gets the same structure as above PLUS these additional sections appended after `## Skills`:

```markdown
## Team Roster

You are the lead agent. Here are your available teammates:

| Agent | Role | When to Spawn |
|-------|------|---------------|
| {name} | {one-line description} | {brief trigger description} |
| ... | ... | ... |

## Routing Guide

When the user's request involves a specific domain, spawn the appropriate teammate:

{For each routing rule derived from bindings[] and AGENTS.md cross-references:}
- **{domain/topic}** -- Spawn **{agent name}** as a teammate. {Brief explanation of why.}

When unsure which agent to spawn, handle the request yourself or ask the user.

## Cross-Cutting Patterns

Some tasks require multiple agents working together:

{For each identified cross-cutting pattern:}
- **{task type}** -- Spawn **{agent A}** and **{agent B}** together. {Explanation.}

## When to Use Teams vs Single Agent

- **Use Agent Teams** (multiple teammates) for: cross-domain tasks, tasks requiring multiple specialties, large coordinated efforts
- **Use single teammate spawn** for: focused tasks within one domain, quick questions, routine work
- **Handle yourself** for: simple questions, routing decisions, status checks, user preferences
```

### 3.3 Create shared skill files

For each multi-agent skill identified in the skill consolidation plan:

```bash
mkdir -p ~/.claude/skills
```

Create `~/.claude/skills/{skill-name}.md`:

```markdown
---
name: {skill-name}
description: {brief description of what the skill does, derived from SKILL.md content}
---

{Content from the original SKILL.md}
```

### 3.4 Seed agent memory

For each agent that has MEMORY.md content:

```bash
mkdir -p ~/.claude/agent-memory/{name}
```

Create `~/.claude/agent-memory/{name}/MEMORY.md` with the content from the agent's OpenClaw MEMORY.md.

Do NOT copy daily notes (`memory/YYYY-MM-DD.md`) -- these are ephemeral by design.

### 3.5 Print post-migration summary

After all artifacts are generated, print this summary:

```
## Migration Complete

### Generated Artifacts
- **X agent definitions** created in ~/.claude/agents/
  {list each: ~/.claude/agents/{name}.md}
- **Y shared skills** created in ~/.claude/skills/
  {list each: ~/.claude/skills/{name}.md}
- **Z agent memory files** seeded in ~/.claude/agent-memory/
  {list each: ~/.claude/agent-memory/{name}/MEMORY.md}

### Next Steps

1. **Enable Agent Teams:**
   Add this to your `~/.claude/settings.json` under the `env` key:
   ```json
   {
     "env": {
       "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
     }
   }
   ```

2. **Configure MCP Servers:**
   {For each MCP server that needs credentials:}
   - {server name}: Set up {credential type} in {location}

3. **Verify agents loaded:**
   Run `/agents` in Claude Code to see all migrated agents.

4. **Try your first Agent Team:**
   > Create an agent team with {2-3 agent names from the migration} to {sample task based on their roles}

5. **Migration plan for reference:**
   ~/claw-to-claude-migration-plan.md
```

---

## Error Handling

Apply these rules throughout all phases:

| Situation | Action |
|-----------|--------|
| Workspace file missing (no SOUL.md, etc.) | Skip that section in the agent body. Note the gap in the plan. |
| Agent has no workspace files at all | Create agent with frontmatter only, empty body with a comment: "No workspace files found -- add agent instructions here." |
| openclaw.json cannot be found | Report error, print expected locations, stop. |
| openclaw.json cannot be parsed (invalid JSON/JSON5) | Report the parse error with the problematic line if possible, stop. |
| SSH connection fails | Suggest --path approach with SCP instructions (see Step 0). |
| Existing CC agent file at target path | Ask user: overwrite / skip / merge (see step 3.0). |
| Unknown MCP server type | Include in "What Won't Migrate" section with the raw config for reference. |
| Non-Anthropic model (openai/*, etc.) | Flag in plan, default to `inherit`, ask user to confirm. |
| Skill directory exists but SKILL.md is missing | Skip the skill, note it as a gap. |
| Circular delegation references | Note in plan, suggest user review the routing guide manually. |
