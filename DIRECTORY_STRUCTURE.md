# Roachie / Roachman — Directory Structure

```
roachie/
├── bin/                              # Entry points
│   ├── roachman                      #   Main CLI (~2,750 lines, 13 menus)
│   ├── roachie-nl                    #   Natural language interface (interactive)
│   ├── roachie-batch                 #   Batch/scripted NL interface
│   └── roachman-bash.sh              #   Bash completion helper
│
├── src/lib/                          # Sourced modules (18 files)
│   ├── cluster.sh                    #   Cluster create/destroy/status
│   ├── replication.sh                #   Physical cluster replication, failover
│   ├── upgrade.sh                    #   Rolling upgrades, semver parsing
│   ├── haproxy.sh                    #   HAProxy load balancer management
│   ├── kafka.sh                      #   Kafka integration
│   ├── ceph.sh                       #   Ceph/S3GW storage integration
│   ├── monitoring.sh                 #   Prometheus + Grafana setup
│   ├── ports.sh                      #   Centralized port config (PORT_CONFIG)
│   ├── sql.sh                        #   SQL connection wrapper
│   ├── validation.sh                 #   Input validation helpers
│   ├── errors.sh                     #   error(), warning(), info(), success()
│   ├── llm.sh                        #   LLM module loader
│   ├── llm_init.sh                   #   LLM initialization
│   ├── llm_prompt.sh                 #   Prompt construction, semantic matching
│   ├── llm_providers.sh              #   API calls (Anthropic/OpenAI/Gemini/Vertex/Ollama)
│   ├── llm_assistant.sh              #   Command extraction, validation, execution
│   ├── llm_metrics.sh                #   Metrics logging, persistent learning
│   └── llm_voice.sh                  #   Voice input (Whisper)
│
├── tools/
│   ├── current -> tools.25.2         #   Symlink to active version
│   ├── tools.25.2/                   #   77 cr_* tools (bash scripts)
│   │   ├── cr_backup                 #     (77 tools: cr_backup .. cr_workload)
│   │   ├── cr_create_cluster         #     Standalone cluster creation
│   │   ├── cr_remove_cluster         #     Standalone cluster removal
│   │   ├── cr_upgrade_version        #     Standalone version upgrade
│   │   └── README_V25.2.md           #     Schema compatibility docs
│   ├── lib/
│   │   └── common.sh                 #   Shared library for all cr_* tools
│   ├── embeddings/                   #   Pre-computed embeddings (3 providers)
│   │   ├── gemini_gemini-embedding-001.json
│   │   ├── ollama_nomic-embed-text.json
│   │   └── openai_text-embedding-3-small.json
│   └── utils/
│       ├── tool_descriptions.txt     #   77 curated tool descriptions
│       ├── generate_embeddings.sh    #   Embedding generator (--provider)
│       ├── test_embeddings.sh        #   Accuracy test suite (135 queries)
│       ├── Modelfile.roachie*        #   Ollama model definitions (3 files)
│       └── sample_schema*.sql        #   Sample schemas for demos
│
├── tests/
│   ├── run-all-tests.sh              #   Main test runner
│   ├── test-all-tiers.sh             #   Tiered runner (--quick/--all/--llm)
│   ├── unit/                         #   20 unit test suites (~550 assertions)
│   │   ├── test_helpers.sh           #     Test framework (assertions)
│   │   ├── test_validation.sh        #     Input validation tests
│   │   ├── test_pure_functions.sh    #     Semver, ports, naming
│   │   ├── test_llm_*.sh            #     LLM module tests (5 files)
│   │   ├── test_kafka.sh            #     Kafka module tests
│   │   └── ...                       #     (20 files total)
│   ├── integration/                  #   9 integration suites (cluster required)
│   │   ├── test_demo_1_full_lifecycle.sh
│   │   ├── test_demo_2_a_to_b.sh
│   │   └── ...                       #     (9 files total)
│   ├── manual/                       #   12 manual test scripts
│   └── fixtures/                     #   Test data
│
├── config/
│   └── roachman.conf.example         #   Configuration template
│
├── docs/                             #   Documentation
│   ├── architecture/                 #     Architecture docs
│   ├── diagrams/                     #     Architecture diagrams
│   ├── getting-started/              #     Onboarding guides
│   ├── guides/                       #     How-to guides
│   ├── reference/                    #     Reference docs
│   ├── testing/                      #     Test documentation
│   ├── tools/                        #     Tool-specific docs
│   └── troubleshooting/              #     Troubleshooting guides
│
├── examples/
│   ├── queries/                      #   Example NL queries (6 files)
│   ├── docker/docker-compose.yml     #   Docker Compose example
│   ├── kubernetes/                   #   K8s examples
│   └── systemd/                      #   Systemd service examples
│
├── integrations/
│   └── slack/                        #   Slack bot integration
│       ├── roachman_slack_bot.py
│       ├── QUICKSTART.md
│       └── ...
│
├── dev/scripts/                      #   Development scripts
│   ├── demo/                         #     Demo automation
│   ├── improvements/                 #     Code improvement scripts
│   ├── setup/                        #     Installation helpers
│   ├── maintenance/                  #     Maintenance utilities
│   └── testing/manual/               #     Manual test scripts
│
├── data/                             #   Runtime data (cache, audit, tmp)
├── sql_audit/                        #   NL query audit logs (CSV/JSONL)
├── archive/                          #   Archived backups & old scripts
├── .github/workflows/tests.yml       #   CI pipeline
├── CLAUDE.md                         #   Project instructions for Claude
└── README.md                         #   Project README
```

## Key Stats

- **77** cr_* CLI tools for CockroachDB administration
- **18** sourced shell modules in `src/lib/`
- **20** unit test suites (~550 assertions)
- **9** integration test suites (cluster required)
- **3** embedding providers (Gemini, OpenAI, Ollama)
- **5** LLM providers supported (Anthropic, OpenAI, Gemini, Vertex AI, Ollama)
