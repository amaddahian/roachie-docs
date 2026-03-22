# Session Changelog — 2026-03-21

## Summary

This session focused on three areas: (1) diagnosing and improving Ollama/Llama accuracy for the NL interface, (2) testing a few-shot examples approach, and (3) setting up Claude Code project infrastructure following best practices.

---

## 1. Ollama vs Gemini Accuracy Gap — Root Cause & Fix

### Problem
Ollama (roachie-8b, Llama 3.1 8B) achieved only 72-78% pass rate on 73-prompt batch tests, while Gemini consistently hit 95-100% on the same prompts.

### Root Cause Analysis (3 Compounding Factors)

| Factor | Impact | Description |
|--------|--------|-------------|
| Prompt divergence | ~11% accuracy loss | Ollama used `_nl_build_system_prompt_compact()` which skipped 7 template files (~6,470 tokens of critical tool documentation). Gemini used the full `_nl_build_system_prompt()`. |
| Enrichment throttling | ~3-5% accuracy loss | Ollama was limited to 3 enriched tools vs 7 for Gemini (`_NL_MAX_ENRICHED_TOOLS`). Less `--help` context meant worse parameter selection. |
| Model capability ceiling | Fundamental | Even with identical prompts and enrichment, 8B model can't match Gemini. Verified by running 12 failed queries through both with identical enrichment — Gemini: 12/12, Ollama: 2/12. |

### Key Insight: Ollama Tokenizer Efficiency
Ollama's llama tokenizer is ~3x more efficient than Gemini's. 4K Ollama tokens = ~12K Gemini tokens in actual text content. This meant the compact prompt was an unnecessary restriction — the full prompt fits comfortably within expanded context.

### Fix Applied (Commit `3808908`)

**Unified prompt path** — removed all Ollama-specific branching:

- `src/lib/llm_prompt.sh`: Removed `_nl_build_system_prompt_compact()` branch; all providers now use `_nl_build_system_prompt()`
- `bin/roachie-batch`: Removed Ollama enrichment limit (`_enrich_max=3`) and separate connection reminder; all providers use same enrichment and `_nl_connection_reminder()`
- `src/lib/llm_config.sh`: Increased Ollama context from 8192 to 16384 tokens
- `tools/utils/Modelfile.roachie-8b`, `Modelfile.roachie`, `Modelfile.roachie-3b`: Changed `PARAMETER num_ctx` from 8192 to 16384

**Result**: Ollama accuracy improved from 72% to 83% (+11 points).

### Historical Accuracy (Debunked 100% Myth)
Analysis of all batch test history in `nl_batch_summary.csv` proved Ollama **never exceeded 78%** on full 73-prompt live runs. The 97% result was DRY-RUN mode. The 100% runs were always Gemini.

---

## 2. Few-Shot Examples Approach

### Attempt 1: Verbose Examples (775 tokens)
- Included CORRECT/WRONG pairs with explanations
- **Caused regression**: 83% dropped to 76% (56/73), plus 2 new `invalid_json` errors
- Root cause: "lost-in-the-middle" effect — 8B model gets overwhelmed with too much prompt content

### Attempt 2: Compact Examples (220 tokens) — Commit `cd9c457`
- Single-line patterns only, no WRONG examples, no explanations
- Created `tools/utils/prompts/few_shot_examples.txt` (882 chars)
- Loaded at end of `_nl_build_system_prompt()` via `_nl_load_prompt_template`
- **Result**: Neutral (82-83%), fixed 3 specific failures (#23 DDL schema, #32 DDL users, #38 DDL views) but model still ignores examples for other patterns

### Current Few-Shot Content
```
EXAMPLES OF CORRECT COMMANDS:
- DDL for table: cr_ddl_table -d DB rides -t TENANT --insecure (table name is positional, NOT -T)
- DDL for views: cr_ddl_view -d DB -t TENANT --insecure (NO --all-views flag)
- DDL for users: cr_ddl_user -t TENANT --insecure (dedicated tool, NOT cr_ddl --include-users)
- DDL for schema: cr_ddl -d DB -s SCHEMA --all -t TENANT --insecure
- Columns: cr_columns --filter-db DB --filter-table TABLE -t TENANT --insecure (NOT -T)
- ACL: cr_get_acl -d DB -o OBJECT -t TENANT --insecure (use -o, NOT -T)
- Ceph backups: cr_ceph_storage list-backups (NOT "list"), show-backup, backup -d DB --bucket NAME
- Search tools: cr_help --search "keyword" (NO connection flags)
- Migrate: cr_migrate --source-host H --source-port P --source-db DB ...
```

---

## 3. Claude Code Project Infrastructure — Commit `a732a43`

### Assessment Against Best Practices
Compared roachie project setup against the "4-layer framework" for Claude Code projects (CLAUDE.md, Skills, Hooks, Agents). Results saved in `analysis-claude-code-project-setup.md`.

### Slash Commands Created (`.claude/commands/`)

| Command | Purpose |
|---------|---------|
| `batch-test.md` | Run NL batch tests with configurable provider, prompt count, and tools version |
| `test.md` | Run unit/integration/LLM test suites |
| `check.md` | Run ShellCheck on modified or specified files |
| `review.md` | Full review workflow: shellcheck + tests + optional batch test |
| `compare-providers.md` | Side-by-side Ollama vs Gemini accuracy comparison |

### Hooks Added (`.claude/settings.local.json` — local only)
- **PreToolUse hook on `git commit`**: Automatically runs ShellCheck on staged `.sh` files before allowing commit. Blocks commit if warnings found.

### .gitignore Updated
```
# Claude — ignore local settings, keep shared commands
.claude/
!.claude/commands/
```

---

## Commits (all pushed to origin/main)

| Commit | Description |
|--------|-------------|
| `3808908` | Unify Ollama prompt path with cloud providers (72% -> 83%) |
| `cd9c457` | Add compact few-shot examples to system prompt |
| `a732a43` | Add Claude Code slash commands for common workflows |

---

## Current Baselines

| Provider | Pass Rate | Notes |
|----------|-----------|-------|
| Gemini | 95-100% | Full prompt, 7 enriched tools |
| Ollama (roachie-8b) | 82-83% | Same prompt as Gemini, 16K context, 8B model ceiling |

## Potential Next Steps (Not Started)

- Larger Ollama model (70B) — would need more GPU memory
- Fine-tuning Llama on Gemini outputs — curated training set from successful Gemini responses
- Accept 83% as the practical ceiling for 8B parameter models

---

## Related Documents
- `analysis-ollama-vs-gemini-accuracy.md` — Detailed accuracy analysis with token budgets and command comparisons
- `analysis-claude-code-project-setup.md` — Roachie vs Claude Code 4-layer framework comparison
- `ollama-chunking-strategies.md` — Previous chunking strategies (now partially superseded by unified prompt)
