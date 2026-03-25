# Model Comparison: Failure Analysis (2026-03-24)

## Test Configuration
- **Query file**: `examples/queries/example-queries.txt` (69 prompts)
- **Tools version**: 25.4
- **Flags**: `--no-feedback --no-followup --allow-sql-files`
- **All three runs performed back-to-back on the same machine**

## Summary

| Model | Base | Size | Fine-Tuned | Pass | Fail | Skip | Rate | Avg API ms |
|-------|------|------|------------|------|------|------|------|------------|
| **roachie-8b** | Llama 3.1 8B Q4 | 4.9 GB | No | 65 | 4 | 0 | **94%** | 22,434 |
| **roachie-nemo-ft** | Mistral Nemo 12B Q8 | 13 GB | Yes (LoRA) | 62 | 7 | 0 | **89%** | 38,014 |
| **roachie-8b-ft** | Llama 3.1 8B FP16 | 16 GB | Yes (LoRA) | 61 | 7 | 1 | **88%** | 28,086 |

### Key Finding
The **base model (roachie-8b) outperforms both fine-tuned models** in this run at 94% vs 88-89%. This is a reversal from the 03-22 results where fine-tuning improved accuracy from 82% to 90%. The improvement is likely due to incremental prompt/template improvements made between 03-22 and 03-24 that benefit the base model equally.

## Failure Matrix

| # | Prompt | 8b | 8b-ft | nemo-ft |
|---|--------|:---:|:-----:|:-------:|
| 21 | generate statistics on all tables in the movr database... | OK | OK | FAIL |
| 32 | list all groups associated with test_lab database... | OK | FAIL | OK |
| 38 | show me tool names including table keyword | FAIL | FAIL | FAIL |
| 39 | show me tool names including view keyword | FAIL | SKIP | FAIL |
| 42 | check table statistics for movr database... auto-generate | OK | FAIL | OK |
| 53 | what's the ddl for the products table in test_lab... | FAIL | FAIL | FAIL |
| 60 | show me the top 3 longest running queries... | OK | OK | FAIL |
| 62 | list all unique event types in the last 48 hours... | OK | FAIL | OK |
| 63 | migrate everything from db1 to db2... | OK | OK | FAIL |
| 64 | migrate from inventory_schema in test_lab to db3... | OK | FAIL | OK |
| 66 | Dump out the SQL/DDL for table rides in movr... | FAIL | FAIL | FAIL |

**Universal failures** (all 3 models): #38, #53, #66
**Only base model passes**: #32 (8b-ft fails), #42 (8b-ft fails), #62 (8b-ft fails), #64 (8b-ft fails)
**Only fine-tuned models fail**: #21 (nemo-ft), #60 (nemo-ft), #63 (nemo-ft)

## Detailed Failure Analysis

### Universal Failures (all 3 models fail)

#### #38: "show me tool names including table keyword"
**Root cause**: Meta-query — user is asking about tool names, not asking to run a tool.
- **8b**: Lists 7 table-related tools with `<database>` placeholders → BLOCKED-METACHAR
- **8b-ft**: Lists 4 table-related tools with `<database>` `<host>` placeholders → BLOCKED-METACHAR
- **nemo-ft**: Tries `cr_help --search "table"` with connection flags → FAILED (cr_help doesn't take -h/-p)
- **Best attempt**: nemo-ft (correct tool, wrong flags). Fix: teach `cr_help --search` needs NO connection flags.

#### #53: "what's the ddl for the products table in test_lab"
**Root cause**: `cr_ddl_table` requires table name as positional arg, all models get it wrong.
- **8b**: `cr_ddl_table -d test_lab` — missing table name entirely
- **8b-ft**: `cr_ddl_table -d test_lab` — missing table name entirely
- **nemo-ft**: `cr_ddl_table -d test_lab -T products` — uses `-T` flag (doesn't exist, table is positional)
- **Correct**: `cr_ddl_table -d test_lab products -h ... -t ... --insecure`

#### #66: "Dump out the SQL/DDL for table rides in movr"
**Root cause**: Same as #53 — `cr_ddl_table` table name positioning.
- **8b**: `cr_ddl_table -d movr` — missing table name
- **8b-ft**: `cr_ddl_object -d movr -T rides` — wrong tool name + wrong flag
- **nemo-ft**: `cr_ddl_table -d movr -T rides` — right tool, wrong flag
- **Correct**: `cr_ddl_table -d movr rides -h ... -t ... --insecure`

### Fine-tuned-only Failures

#### #21: "generate statistics on all tables in movr" (nemo-ft only)
- **nemo-ft**: `cr_genstats -d movr --execute --all-tables` — invented `--all-tables` flag
- **8b**: Used `cr_table_analyze` instead (different valid approach)
- **8b-ft**: `cr_genstats -d movr --execute` (correct, no `--all-tables`)

#### #32: "list all groups associated with test_lab" (8b-ft only)
- **8b-ft**: `cr_get_acl -d test_lab` — wrong tool (ACL, not groups)
- **8b**: `cr_ddl_group -d test_lab` (correct)
- **nemo-ft**: `cr_db_group_access_listing -d test_lab` (correct)

#### #42: "check table statistics with auto-generate" (8b-ft only)
- **8b-ft**: `cr_check_statistics --auto-generate` — missing `-d movr`
- **8b**: `cr_check_statistics -d movr --auto-generate` (correct)
- **nemo-ft**: Same as 8b (correct)

#### #60: "top 3 longest running queries" (nemo-ft only)
- **nemo-ft**: First command OK, but adds `cr_plan -q <QUERY_ID>` → BLOCKED-METACHAR
- Other models just run `cr_query_stats` alone (correct — single command is sufficient)

#### #62: "list unique event types in last 48 hours" (8b-ft only)
- **8b-ft**: `--event-type config, restart, ddl, permissions` — space after comma breaks parsing
- **8b**: `--event-type all` (correct)
- **nemo-ft**: `--event-type all` (correct)

#### #63: "migrate from db1 to db2" (nemo-ft only)
- **nemo-ft**: Uses `-t system` instead of `--source-tenant system --target-tenant system`
- Both Llama models get the full flag syntax correct

#### #64: "migrate from inventory_schema to db3" (8b-ft only)
- **8b-ft**: Missing `--source-schema inventory_schema` — ignored the schema part of the prompt
- **8b**: Includes `--source-schema inventory_schema` (correct)
- **nemo-ft**: Includes it plus extra flags (correct)

## Failure Categories

| Category | Count | Prompts |
|----------|-------|---------|
| Positional arg confusion (`cr_ddl_table TABLE`) | 3 universal | #53, #66 |
| Meta-query (asking about tools, not running them) | 3 universal | #38, #39 |
| Invented/wrong flags | 3 | #21, #42, #62 |
| Missing required flags | 2 | #32, #63 |
| Over-generation (adding unnecessary 2nd command) | 1 | #60 |
| Wrong tool selection | 1 | #32 |
| Missing parameter from prompt | 1 | #64 |

## Recommendations

1. **`cr_ddl_table` positional arg**: Add explicit few-shot example to `few_shot_examples.txt`:
   `cr_ddl_table -d DB TABLE_NAME -t TENANT --insecure` (table name is positional after -d DB)
2. **`cr_help --search`**: Reinforce in few-shot that this takes NO connection flags (-h, -p, -t)
3. **Meta-queries**: Consider adding a "tool discovery" instruction — when user asks about tool names, use `cr_help --search`
4. **`cr_migrate` flags**: The fine-tuned models sometimes use `-t` shorthand instead of `--source-tenant`/`--target-tenant` — add to few-shot examples
