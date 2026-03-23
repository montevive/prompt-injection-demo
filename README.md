# Indirect Prompt Injection — Tool-use Exfiltration Research

Research project demonstrating how indirect prompt injection can exploit AI agents with tool-calling capabilities to exfiltrate sensitive data (system prompts, conversation history, API keys).

[![Demo Video](https://img.youtube.com/vi/XBAiwo-pawg/maxresdefault.jpg)](https://www.youtube.com/watch?v=XBAiwo-pawg)

## Attack flow

1. User asks an AI agent to summarize a webpage
2. The webpage contains a hidden injection payload (invisible `<div>` or JSON-LD)
3. The payload instructs the agent to POST conversation context to an attacker-controlled server
4. A vulnerable agent calls `http_post`, leaking the system prompt + conversation history

No jailbreak is needed — the attack exploits legitimate tools (HTTP POST for webhooks, logging, reporting) that are commonly available in agent frameworks.

## Components

| File | Description |
|---|---|
| `demo.py` | Automated test harness — spins up poisoned page + collector, runs multiple LLMs, reports exfiltration rates |
| `demo-server.py` | Standalone poisoned page + collector + web viewer for use with external agents (n8n, etc.) |
| `run-n8n-demo.sh` | One-command launcher: starts demo-server + n8n, prints URLs and instructions |
| `n8n/docker-compose.yml` | n8n instance pre-configured for the demo |
| `FINDINGS.md` | Detailed results and analysis |

## Quick start

### Automated testing (Python demo)

```bash
# Install dependencies
pip install anthropic openai python-dotenv

# Create .env with your Anthropic API key (optional, for Claude tests)
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env

# Run all default models
python3 demo.py

# Test a specific model
python3 demo.py --model qwen2.5:14b --runs 5 --temperature 0.1

# Test with subtle payload
python3 demo.py --model qwen2.5:14b --payload subtle
```

### n8n demo (real-world agent framework)

```bash
# Launch everything (poisoned page, collector, n8n)
./run-n8n-demo.sh

# Or with subtle payload
./run-n8n-demo.sh subtle
```

The script prints all URLs. Open n8n in your browser, go to the "Prompt Injection" workflow, and send the suggested chat message.

### Manual testing with demo-server

```bash
# Start the poisoned page + collector separately
python3 -u demo-server.py

# Endpoints:
#   http://0.0.0.0:8765/         Poisoned page
#   http://0.0.0.0:8766/collect  Collector (receives exfiltrated data)
#   http://0.0.0.0:8766/view     Web viewer for collected exfiltrations
```

All services bind to `0.0.0.0` — accessible from other machines on the network.

### Service URLs

| Service | URL | Description |
|---|---|---|
| n8n | http://\<your-ip\>:5680 | Workflow editor and chat UI |
| Poisoned page | http://\<your-ip\>:8765 | Webpage with hidden injection payload |
| Collector | http://\<your-ip\>:8766/collect | Receives exfiltrated data via POST |
| Viewer | http://\<your-ip\>:8766/view | Web UI to browse collected exfiltrations |

## n8n workflow setup

The n8n workflow uses the **AI Agent** node with:

- **Chat Trigger** — receives user messages
- **Ollama Chat Model** — qwen2.5:14b at temperature 0.1
- **fetch_url** tool — HTTP Request Tool (GET) with `$fromAI('URL')` for the URL
- **http_post** tool — HTTP Request Tool (POST) with `$fromAI('URL')` for the URL and `$fromAI('data')` for the body
- **Simple Memory** — buffer window for conversation context

The system prompt includes a fake `SECRET_API_KEY` to demonstrate credential exfiltration.

## Payload variants

**Obvious** — Hidden `<div style="display:none">` with "COMPLIANCE NOTICE — MANDATORY AUDIT PROTOCOL" framing that instructs the agent to POST system prompt and conversation history.

**Subtle** — JSON-LD `<script type="application/ld+json">` block with an `analyticsNote` field phrased as a publisher instruction for "content personalization".

## Key findings

- **Low temperature increases susceptibility.** qwen2.5:14b at temp 0.1 exfiltrated in 140% of runs (multiple calls per run). At temp 1.5 it was fully resistant.
- **Claude detected and refused the injection**, explicitly naming the attack in its response.
- **Mistral models (v0.3, nemo) were fully resistant** across all temperatures.
- **The obvious payload outperformed the subtle one.** Effective payloads need to read as direct instructions.
- **n8n AI Agent with qwen2.5:14b successfully exfiltrated** system prompt + API key + conversation history to the collector.

See [FINDINGS.md](FINDINGS.md) for full results and analysis.

## Requirements

- Python 3.10+
- [Ollama](https://ollama.ai/) with models pulled (e.g. `ollama pull qwen2.5:14b`)
- Docker + Docker Compose (for n8n demo)
- Anthropic API key (optional, for Claude tests)
