# Claude Code Project Setup: Roachie vs Best Practices — 2026-03-21

## Source

LinkedIn post on organizing Claude Code projects (4-layer framework):
https://www.linkedin.com/feed/update/urn:li:activity:7440971600608362496/

## The 4-Layer Framework (from the post)

1. **CLAUDE.md** — persistent project memory (WHAT/WHY/HOW framework)
2. **Skills** — auto-activated knowledge packages with SKILL.md descriptors
3. **Hooks** — enforced safety gates (PreToolUse/PostToolUse), not suggestions
4. **Agents** — subagent definitions in .yml with inherited context boundaries

### Recommended File Structure

```
project/
├── CLAUDE.md              (Project memory with WHAT/WHY/HOW)
├── .claude/               (Extensions hub)
├── commands/              (Slash commands as .md files)
├── skills/                (Auto-triggered workflows)
│   └── SKILL.md
├── agents/                (Subagent definitions in .yml)
├── plugins/               (Distributable setups)
└── .mcp.json             (External tool connections)
```

### Key Recommendations

- "If you're using Claude Code without setting up CLAUDE.md and skills — you're leaving 80% of its power on the table"
- Context budget thresholds (0-60%/50-70%/70-80%/80%+) require hook enforcement
- Consistent team setup prevents compounding technical debt
- Daily workflow: Plan Mode → Auto-Accept → commit loop
- Hook events enforce conventions across team environments
- Skills function like microservices with defined interfaces
- Evaluation and feedback loops for systematic testing

## How Roachie Compares

### 1. CLAUDE.md — Project Memory ✅ Strong (Grade: A)

The post recommends a WHAT/WHY/HOW framework. Roachie has this covered well:

| Recommendation | Roachie Status |
|---|---|
| Project overview & architecture | ✅ Detailed in `CLAUDE.md` (architecture, key files table, env vars) |
| Code conventions | ✅ ShellCheck rules, variable patterns, commit style, error handling |
| Testing instructions | ✅ Exact test commands, version pinning (`CRDB_VERSION=v25.2.12`) |
| Key decisions documented | ✅ Documentation split (roachie-docs repo), review workflow |

**What roachie adds beyond the recommendation**: A separate auto-memory layer (`~/.claude/projects/.../memory/MEMORY.md`) that accumulates cross-session learnings — semantic matching accuracy, fix histories, model quirks. This goes beyond static CLAUDE.md into living project memory.

### 2. Skills — Auto-Triggered Workflows ⚠️ Not Used (Grade: D)

The post recommends `.claude/commands/` with slash commands and `skills/` with SKILL.md descriptors. Roachie doesn't use these at all. Potential opportunities:

- `/batch-test` — could wrap the common `roachie-batch` invocation
- `/review` — could automate the review workflow (implement → test → shellcheck → update docs)
- `/compare-providers` — could run the same prompts through Ollama and Gemini side-by-side

### 3. Hooks — Safety Gates ⚠️ Not Used (Grade: D)

The post emphasizes hooks as **enforced conventions, not suggestions**. Roachie has no `.claude/hooks/` configured. Potential uses:

- **PreToolUse**: Block commits without running `tests/run-all-tests.sh` first
- **PreToolUse**: Enforce `shellcheck -x` on any modified `.sh` file before commit
- **PostToolUse**: Auto-add modified files to a "files changed this session" tracker
- **Context budget enforcement**: The post mentions thresholds (60%/70%/80%) — hooks could warn when approaching limits

### 4. Agents / Subagents ⚠️ Not Used (Grade: D)

The post recommends `agents/*.yml` for subagent definitions. Roachie doesn't define any, though we do use the built-in Agent tool ad-hoc for codebase exploration.

### 5. MCP Integration ✅ Partial (Grade: C)

Roachie has `tools/mcp/tools.yaml` (39 MCP-compatible tools via Google GenAI Toolbox), but no `.mcp.json` in the project root for Claude Code's native MCP integration.

### 6. Daily Workflow (Grade: C)

The post recommends a structured Plan → Auto-Accept → Commit loop. Roachie development is ad-hoc — no structured workflow enforced.

## Overall Assessment

| Layer | Post Recommendation | Roachie | Grade |
|-------|-------------------|---------|-------|
| CLAUDE.md | WHAT/WHY/HOW framework | Comprehensive + auto-memory | **A** |
| Skills | Slash commands, SKILL.md | Not used | **D** |
| Hooks | PreToolUse/PostToolUse gates | Not used | **D** |
| Agents | .yml subagent definitions | Not used | **D** |
| MCP | .mcp.json connections | Partial (toolbox yaml exists) | **C** |
| Daily workflow | Plan → Auto-Accept → Commit | Ad-hoc, no structured loop | **C** |

**Bottom line**: Roachie's CLAUDE.md and memory setup is excellent — arguably better than the post's recommendation with the auto-memory layer. But skills, hooks, and agents are not used at all, which the post claims is "leaving 80% of power on the table."

## Implementation Plan

### Priority 1: Hooks (highest impact — enforced quality gates)
- Pre-commit shellcheck enforcement
- Pre-commit test runner check
- Context budget warnings

### Priority 2: Slash Commands (workflow automation)
- `/batch-test` — run batch tests with common configurations
- `/review` — implement → test → shellcheck → update docs workflow
- `/compare-providers` — side-by-side Ollama vs Gemini comparison

### Priority 3: Agents (specialized subagents)
- Codebase explorer agent
- Test runner agent
- Documentation updater agent
