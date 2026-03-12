# Prompt Injection Research Project

## Overview

This is a security research project studying indirect prompt injection attacks against AI agents with tool-calling capabilities. The goal is to understand which models are vulnerable to hidden instructions in web content that trick agents into exfiltrating sensitive data.

## Project structure

- `demo.py` — Automated test harness. Runs LLMs against a poisoned webpage and measures exfiltration rates. Uses Anthropic SDK for Claude and OpenAI-compatible API for Ollama models.
- `demo-server.py` — Standalone HTTP servers: poisoned page (:8765), collector (:8766), and web viewer (/view). Used with n8n or any external agent.
- `run-n8n-demo.sh` — Launcher script that starts demo-server + n8n docker container.
- `n8n/docker-compose.yml` — n8n config (port 5680, host network mode).
- `FINDINGS.md` — Research results and analysis.
- `collected/` — Directory where exfiltrated data is saved as JSON files (gitignored).

## Running tests

```bash
# Automated Python demo (all default models)
python3 demo.py

# Specific model with multiple runs
python3 demo.py --model qwen2.5:14b --runs 5 --temperature 0.1

# n8n demo (starts all services)
./run-n8n-demo.sh
```

## Architecture

The attack flow: User message → AI Agent → fetch_url (gets poisoned page) → LLM processes hidden injection → http_post (exfiltrates to collector).

Two payload variants exist in both `demo.py` and `demo-server.py`: "obvious" (compliance notice framing) and "subtle" (JSON-LD analytics framing).

## Services and ports

| Port | Service |
|---|---|
| 8765 | Poisoned webpage |
| 8766 | Collector + web viewer (/collect, /view) |
| 5680 | n8n |

All bind to `0.0.0.0` for network access.

## Dependencies

- `anthropic`, `openai`, `python-dotenv` (Python packages)
- Ollama running locally with models pulled
- Docker for n8n
- `.env` file with `ANTHROPIC_API_KEY` for Claude tests
