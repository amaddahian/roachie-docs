# Roachie — New Mac Setup Guide

**Date:** 2026-03-27
**Purpose:** Complete dependency list and installation instructions for setting up roachie on a fresh Mac.

---

## Quick Start (Minimum Viable Setup)

Install the required dependencies, then pick at least one LLM provider:

```bash
# Required tools
brew install bash jq curl

# Container runtime (pick one)
brew install podman
podman machine init --cpus 4 --memory 4096
podman machine start

# CockroachDB CLI
brew install cockroachdb/tap/cockroach

# At least one LLM provider API key in ~/.zshrc
export GEMINI_API_KEY="..."       # Gemini (cheapest, recommended default)
export OPENAI_API_KEY="..."       # OpenAI
export ANTHROPIC_API_KEY="..."    # Anthropic Claude
```

That's enough to run `roachie-nl`, `roachie-batch`, and `roachman` with cloud LLM providers.

---

## Full Dependency Reference

### Table 1: Required Tools

| Tool | Purpose | Install | Verify |
|------|---------|---------|--------|
| bash 4+ | Associative arrays, modern shell features | `brew install bash` | `bash --version` (must be 4+, macOS ships 3.2) |
| jq | JSON parsing (LLM responses, metrics, embeddings) | `brew install jq` | `jq --version` |
| curl | HTTP client for LLM API calls | `brew install curl` | `curl --version` |
| podman or docker | Container runtime for CockroachDB clusters | `brew install podman` | `podman --version` |
| cockroach | CockroachDB CLI for SQL, schema queries | `brew install cockroachdb/tap/cockroach` | `cockroach version` |

### Table 2: Optional Tools

| Tool | Purpose | Install | When Needed |
|------|---------|---------|-------------|
| ollama | Local LLM inference (no API key needed) | `brew install ollama` | Local model inference, embedding generation |
| shellcheck | Shell script linting | `brew install shellcheck` | Running lint tests, `/check` skill |
| gcloud | Google Vertex AI authentication | `brew install google-cloud-sdk` | Using Vertex AI (Claude via Google Cloud) |
| coreutils | gtimeout for command timeouts | `brew install coreutils` | Test suite timeout enforcement |
| python3 | Training data prep, metrics analysis | `brew install python3` | LoRA fine-tuning, voice transcription |
| gh | GitHub CLI | `brew install gh` | Git push/pull, PR creation |
| sox | Audio recording | `brew install sox` | Voice input mode |
| ffmpeg | Audio recording (fallback) | `brew install ffmpeg` | Voice input mode |
| mcp-toolbox | Google GenAI Toolbox for MCP | `brew install mcp-toolbox` | MCP tool integration |
| podman-compose | Multi-container orchestration | `pip3 install podman-compose` | Prometheus+Grafana monitoring stack |
| aws CLI | S3-compatible storage management | `pip3 install awscli` | Ceph/S3GW backup storage |

---

## LLM Provider Setup

At least one provider is needed for the NL interface. Multiple providers enable fallback.

### Table 3: LLM Providers

| Provider | Env Var | Model Default | Cost | Notes |
|----------|---------|---------------|------|-------|
| Gemini | `GEMINI_API_KEY` | gemini-2.5-flash | ~$0.002/query | Recommended default. Cheapest cloud option. |
| OpenAI | `OPENAI_API_KEY` | gpt-4.1 | ~$0.01/query | Also used for OpenAI embeddings |
| Anthropic | `ANTHROPIC_API_KEY` | claude-sonnet-4-20250514 | ~$0.01/query | Direct Anthropic API |
| Vertex AI | `GOOGLE_CLOUD_PROJECT` | claude-sonnet-4@20250514 | ~$0.01/query | Claude via Google Cloud. Requires `gcloud auth login` |
| Ollama | (none — local) | roachie-8b | Free | Requires Ollama + model download |

Add API keys to `~/.zshrc`:

```bash
export GEMINI_API_KEY="your-key-here"
export OPENAI_API_KEY="your-key-here"
export ANTHROPIC_API_KEY="your-key-here"
export GOOGLE_CLOUD_PROJECT="your-project-id"   # For Vertex AI
```

### Table 4: Model Override Env Vars

| Variable | Default | Purpose |
|----------|---------|---------|
| `GEMINI_MODEL` | `gemini-2.5-flash` | Override Gemini model |
| `OPENAI_MODEL` | `gpt-4.1` | Override OpenAI model |
| `VERTEX_CLAUDE_MODEL` | `claude-sonnet-4@20250514` | Override Vertex model |
| `VERTEX_REGION` | `us-east5` | Vertex AI region |

---

## Podman Setup (Container Runtime)

Required for running CockroachDB clusters, HAProxy, Kafka, monitoring stack.

```bash
brew install podman

# Initialize VM — 4 CPUs, 4GB RAM recommended (avoid 2GB — causes hangs)
podman machine init --cpus 4 --memory 4096 --vmtype applehv
podman machine start

# Verify
podman ps
```

**Important:** The default `--memory 2048` with high CPU count causes VM thrashing and `podman ps` hangs. Always use `--memory 4096` or higher.

Docker Desktop is also supported — install via `brew install --cask docker`.

---

## Ollama Setup (Local LLM)

Optional but enables free, private, offline LLM inference.

```bash
# Install
brew install ollama

# Start the server
ollama serve    # or: brew services start ollama

# Pull the embedding model (required for semantic tool matching)
ollama pull nomic-embed-text

# Pull a base LLM model
ollama pull llama3.1

# Build roachie's custom models (optional, for best local accuracy)
cd tools/utils
ollama create roachie-8b -f Modelfile.roachie-8b
```

### Table 5: Ollama Models

| Model | Size | Accuracy | Purpose |
|-------|------|----------|---------|
| nomic-embed-text | 274MB | N/A | Embedding model for semantic tool matching |
| roachie-8b | ~8GB | 97% | Primary local LLM (prompt-tuned Llama 3.1) |
| roachie-3b | ~3GB | ~85% | Lightweight alternative |
| llama3.1 | ~8GB | ~80% | Fallback if roachie models not built |
| roachie-8b-ft | ~8.5GB | 90% | LoRA fine-tuned variant |
| roachie-nemo-ft | ~13GB | 92% | Mistral Nemo LoRA fine-tuned |

Modelfiles for custom models are in `tools/utils/Modelfile.roachie*`.

---

## Container Images

Pulled automatically on first use. No manual pull needed.

### Table 6: Container Images

| Image | Tag | Purpose |
|-------|-----|---------|
| cockroachdb/cockroach | v25.2.12 (testing) | CockroachDB nodes |
| docker.io/apache/kafka | 3.9.0 | Kafka CDC integration |
| docker.io/prom/prometheus | v3.2.1 | Monitoring stack |
| docker.io/grafana/grafana | 11.5.2 | Monitoring dashboards |
| quay.io/s3gw/s3gw | (configurable) | S3-compatible storage (Ceph) |

---

## Python Setup (Fine-Tuning Pipeline)

Only needed for LoRA fine-tuning on Apple Silicon.

```bash
# Create venv
python3 -m venv tools/utils/finetune/.venv
source tools/utils/finetune/.venv/bin/activate

# Install MLX LoRA framework
pip install mlx-lm

# For Mistral GGUF conversion (optional)
git clone https://github.com/ggerganov/llama.cpp /tmp/llama.cpp
cd /tmp/llama.cpp && pip install -r requirements.txt
```

---

## CockroachDB CLI

```bash
# Primary version (matches tools/current → tools.25.2)
brew install cockroachdb/tap/cockroach

# Verify
cockroach version
```

For integration testing, set the version:
```bash
export CRDB_VERSION=v25.2.12
```

---

## Runtime Configuration Env Vars

### Table 7: NL Subsystem Configuration

| Variable | Default | Purpose |
|----------|---------|---------|
| `ROACHIE_BATCH` | `0` | Set to `1` for batch mode (auto-fallback) |
| `ROACHIE_ROLE` | `admin` | RBAC role: admin, dba, analyst, monitor |
| `NL_NATIVE_TOOLS` | `0` | Enable native tool calling via MCP |
| `NL_NO_STREAM` | `0` | Disable streaming output |
| `NL_DEBUG` | unset | Set to `1` for debug logging |
| `NL_MAX_ITERATIONS` | `3` | Max agent loop iterations |
| `NL_REFLEXION` | `true` | Auto-retry with error analysis |
| `NL_HOST` | `localhost` | CockroachDB host (standalone mode) |
| `NL_PORT` | `26257` | CockroachDB port (standalone mode) |
| `NL_TENANT` | `system` | Virtual cluster tenant |
| `NL_INSECURE` | `1` | Use insecure connection |
| `NL_BUDGET` | unset | Session cost budget (e.g., `1.00`) |
| `NL_GEMINI_THINKING` | `0` | Gemini thinking token budget |

---

## Verification

After setup, verify everything works:

```bash
# Unit tests (no cluster needed)
bash tests/run-all-tests.sh

# Quick smoke test with cluster
CRDB_VERSION=v25.2.12 bash tests/test-all-tiers.sh --quick

# NL interface test (requires at least one LLM API key)
echo "show databases" | bin/roachie-batch --provider gemini

# Generate doc embeddings (if Ollama installed)
ollama serve &
bash tools/utils/generate_doc_embeddings.sh --provider ollama
```

---

## CLI Summary

```
Table 8: Setup Checklist

Step  Action                                          Required?
────  ────────────────────────────────────────────────  ─────────
 1    brew install bash jq curl                        Yes
 2    brew install podman (+ machine init 4CPU/4GB)    Yes
 3    brew install cockroachdb/tap/cockroach            Yes
 4    Add LLM API key(s) to ~/.zshrc                   Yes (1+)
 5    brew install gh && gh auth login                  Recommended
 6    brew install shellcheck                           Recommended
 7    brew install coreutils                            Recommended
 8    brew install ollama + pull nomic-embed-text       Optional
 9    brew install python3 + venv + mlx-lm              Optional
10    brew install sox                                  Optional
```
