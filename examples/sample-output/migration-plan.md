# OpenClaw → Claude Code Migration Plan

**Generated:** 2026-04-06  
**Source snapshot:** `~/openclaw-snapshot`  
**OpenClaw agents found:** 4  
**Skills found:** 6  
**MCP servers found:** 2  
**Cron jobs found:** 3  

> Review this document before running Phase 3. Edit any section to adjust the migration. When ready, say **"execute"** to generate all agent files.

---

## 1. Agent Mapping Table

| OpenClaw Agent | Role | CC Agent File | Model | Memory | Notes |
|---|---|---|---|---|---|
| `main` (default) | Chief of staff / CTO | `~/.claude/agents/cto.md` | `opus` | `user` | Becomes lead agent with routing guide |
| `backend` | Backend engineer | `~/.claude/agents/backend-engineer.md` | `sonnet` | `user` | — |
| `frontend` | Frontend engineer | `~/.claude/agents/frontend-engineer.md` | `sonnet` | `user` | — |
| `analyst` | Data analyst | `~/.claude/agents/data-analyst.md` | `sonnet` | `user` | — |

**Tool mappings:**

| Agent | CC `tools` | CC `disallowedTools` | MCP Servers |
|---|---|---|---|
| `cto` | Read, Grep, Glob, Bash, Write, Edit, WebSearch, WebFetch, Task | — | — |
| `backend-engineer` | Read, Grep, Glob, Bash, Write, Edit | WebSearch, WebFetch | `postgres` |
| `frontend-engineer` | Read, Grep, Glob, Bash, Write, Edit, WebSearch | — | — |
| `data-analyst` | Read, Grep, Glob, Bash | Write, Edit | `postgres` |

---

## 2. Org Hierarchy & Lead

**Lead agent:** `cto` (mapped from OpenClaw default agent `main`)

**Delegation chain:**
```
cto (lead)
├── backend-engineer   — server, DB, integrations
├── frontend-engineer  — UI, React, client-side
└── data-analyst       — metrics, SQL, reporting
```

**Routing logic for lead agent body:**

| User asks about… | Spawn |
|---|---|
| API endpoints, DB schema, background jobs | `backend-engineer` |
| UI components, pages, frontend bugs | `frontend-engineer` |
| Metrics, reporting, data quality | `data-analyst` |
| Full feature (frontend + backend) | `frontend-engineer` + `backend-engineer` as team |
| Security or auth | Handle as `cto`; no security specialist in this setup |

**Note:** No `security-reviewer` agent was found in the OpenClaw setup. The CTO agent will handle security questions directly. Consider adding one post-migration.

---

## 3. Skill Consolidation

Skills found in workspace:

| Skill | Used by agents | Disposition |
|---|---|---|
| `database-query-advisor` | `backend`, `analyst` | **Shared** → `~/.claude/skills/database-query-advisor.md` |
| `pr-review-checklist` | `backend`, `frontend` | **Shared** → `~/.claude/skills/pr-review-checklist.md` |
| `api-contract-template` | `backend` only | **Embedded** → merged into `backend-engineer.md` body |
| `component-design-guide` | `frontend` only | **Embedded** → merged into `frontend-engineer.md` body |
| `metric-naming-convention` | `analyst` only | **Embedded** → merged into `data-analyst.md` body |
| `incident-runbook` | `main` only | **Embedded** → merged into `cto.md` body |

Shared skills will be referenced in frontmatter:
```yaml
skills:
  - database-query-advisor
  - pr-review-checklist
```

---

## 4. MCP Server Config

**2 MCP servers found in OpenClaw config:**

### postgres (per-agent)

- **Type:** stdio
- **Scope:** Per-agent (only `backend` and `analyst` agents have access)
- **Placement:** `mcpServers` frontmatter in each agent file (not global `settings.json`)
- **Config to use:**
  ```yaml
  mcpServers:
    postgres:
      command: npx
      args: ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/acme_dev"]
  ```
- **Action required:** Set `DATABASE_URL` environment variable. The connection string above uses the dev database. Replace with your actual connection string.

### github (global)

- **Type:** stdio
- **Scope:** Global (all agents)
- **Placement:** `~/.claude/settings.json` under `mcpServers`
- **Config to use:**
  ```json
  "github": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-github"],
    "env": {
      "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-token>"
    }
  }
  ```
- **Action required:** Create a GitHub PAT with `repo` and `read:org` scopes and add it to the config.

---

## 5. Scheduled Tasks

**3 cron jobs found in OpenClaw config:**

| OpenClaw Schedule | Target Agent | Prompt | CC Schedule Expression |
|---|---|---|---|
| `0 9 * * 1-5` (weekdays 9am) | `analyst` | "Pull yesterday's signups and activation metrics. Post a 3-bullet summary." | `0 9 * * 1-5` |
| `0 18 * * 5` (Friday 6pm) | `main` | "Review open PRs and any unresolved incidents from this week. Prepare a short EOW summary." | `0 18 * * 5` |
| `*/30 * * * *` (every 30 min) | `backend` | "Check the error rate in the last 30 minutes. Alert if any 5xx rate exceeds 1%." | `*/30 * * * *` |

To register these in Claude Code after migration:
```bash
claude schedule create --cron "0 9 * * 1-5" --agent data-analyst --prompt "Pull yesterday's signups and activation metrics. Post a 3-bullet summary."
claude schedule create --cron "0 18 * * 5" --agent cto --prompt "Review open PRs and any unresolved incidents from this week. Prepare a short EOW summary."
claude schedule create --cron "*/30 * * * *" --agent backend-engineer --prompt "Check the error rate in the last 30 minutes. Alert if any 5xx rate exceeds 1%."
```

---

## 6. What Won't Migrate

The following OpenClaw features have no direct Claude Code equivalent and are excluded from migration:

| Feature | Why it won't migrate | Recommendation |
|---|---|---|
| **Per-agent Telegram bot bindings** | OpenClaw bound each agent to its own Telegram bot token. CC supports a single Telegram channel shared across all agents. | Set up one CC Telegram channel. Messages go to the lead (`cto`) agent, which routes to teammates as needed. |
| **`analyst` agent's Slack channel binding** | CC has no native Slack integration. | Use the Slack MCP server if you need Slack access from agents. Manual alternative: route Slack notifications via a webhook. |
| **Vector memory search** | OpenClaw's `analyst` agent used semantic search over historical daily notes. CC uses file-based grep over `~/.claude/agent-memory/`. | For high-volume memory search, consider an external MCP-accessible vector store (e.g., Pinecone MCP server). |
| **Daily notes** (`memory/YYYY-MM-DD.md`) | Ephemeral by design. Not migrated. | Long-term decisions are already captured in each agent's `MEMORY.md`, which is being migrated. |

---

## 7. Post-Migration Checklist

Complete these steps after Phase 3 executes:

- [ ] **Enable Agent Teams** — Add to `~/.claude/settings.json`:
  ```json
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
  ```

- [ ] **Configure PostgreSQL MCP** — Set your actual connection string in the `mcpServers` config for `backend-engineer.md` and `data-analyst.md`.

- [ ] **Configure GitHub MCP** — Add your GitHub Personal Access Token to `~/.claude/settings.json` under the `github` MCP server entry.

- [ ] **Register scheduled tasks** — Run the three `claude schedule create` commands listed in Section 5.

- [ ] **Set up Telegram channel (optional)** — Run `/telegram:configure` in Claude Code and follow the setup flow. Only one bot token needed now (no per-agent bots).

- [ ] **Verify agents are visible** — Open Claude Code and run `/agents`. You should see: `cto`, `backend-engineer`, `frontend-engineer`, `data-analyst`.

- [ ] **Test a team spawn** — Try:
  > "Create an agent team with backend-engineer and frontend-engineer to design the API contract and UI for a new user profile page."

- [ ] **Confirm memory seeded correctly** — Ask each agent: "What do you know about this project?" Each should recall key decisions from their migrated `MEMORY.md`.
