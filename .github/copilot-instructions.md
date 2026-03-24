# Copilot Instructions — HR Hosted Agent (Vibe Coding)

## Project Overview

This repository is a **template + tutorial** for containerizing a [Microsoft Agent Framework](https://github.com/microsoft/agent-framework) agent and deploying it as a **hosted agent** on [Microsoft Foundry Agent Service](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents).

The included HR agent is a sample — the pattern is designed to be reused for any Agent Framework agent.

## Architecture

```text
ChatAgent → from_agent_framework(agent).run() → HTTP server (:8088)
                                                  ├── POST /responses  (OpenAI Responses API)
                                                  └── GET  /readiness  (health check)
```

## Key Technologies

- **Microsoft Agent Framework** (`agent-framework-core`, `agent-framework-azure-ai`, `agent-framework-azure-ai-search`)
- **Azure AI AgentServer hosting adapter** (`azure-ai-agentserver-agentframework`) — wraps `ChatAgent` into Uvicorn HTTP
- **Azure AI Projects SDK** (`azure-ai-projects`) — registers the container with Foundry
- **Azure Identity** (`azure-identity`) — `DefaultAzureCredential` for auth (sync, not async)
- **OpenTelemetry** — optional tracing to Azure Monitor / Application Insights
- **Docker** — containerization targeting `linux/amd64`
- **Bicep** — infrastructure-as-code for enterprise deployment (`enterprise/infra/`)

## Project Structure

| Path | Purpose |
| ------ | --------- |
| `main.py` | Hosted agent entry point — adapt your agent here |
| `deploy.py` | Registers the container with Foundry via SDK |
| `agent.yaml` | Agent manifest for Foundry |
| `Dockerfile` | Container image definition |
| `requirements.txt` | Python dependencies |
| `original/hr_agent.py` | Original standalone agent (reference only) |
| `enterprise/` | Enterprise variant with CMK, Managed Identity, Private Endpoints |
| `enterprise/infra/` | Bicep modules for enterprise infrastructure |

## Vibe Coding Guidelines

When building or modifying agents in this repo, follow these principles:

### 1. Always Use Microsoft Learn MCP Server

This workspace has the **Microsoft Learn MCP Server** configured (`.vscode/mcp.json`). When working with Azure AI, Agent Framework, or Foundry documentation:

- Use `microsoft_docs_search` to find relevant official docs
- Use `microsoft_code_sample_search` to find code examples
- Use `microsoft_docs_fetch` to get full page content when needed
- **Always ground answers in official Microsoft documentation** rather than guessing API usage

### 2. Agent Adaptation Pattern

To create a new agent from this template:

1. **Replace instructions** — Change `HR_INSTRUCTIONS` in `main.py` with your agent's system prompt
2. **Replace context providers** — Swap `AzureAISearchContextProvider` with your agent's tools, or use `context_providers=[]` if none
3. **Update identity** — Change `name` and `id` in the `ChatAgent` constructor
4. **Update deployment** — Modify `deploy.py` with new agent name, description, and environment variables
5. **Update manifest** — Edit `agent.yaml` with your agent's metadata

### 3. Hosting Constraints (Important!)

- **Must use `ChatAgent`** (not `Agent`) — the hosting adapter requires this class
- **Must use sync `DefaultAzureCredential`** (not async) — the adapter manages async internally
- **Must call `from_agent_framework(agent).run()`** — this starts the HTTP server on port 8088
- **Port 8088** is mandatory — Foundry expects this port
- **Environment variables** for configuration (not `.env` files) — set in `deploy.py` or `agent.yaml`

### 4. Enterprise Considerations

The `enterprise/` folder demonstrates the same agent with enterprise-grade infrastructure:

- **Application code does NOT change** — all security is at the infrastructure level
- **Customer-Managed Keys (CMK)** for encryption
- **Managed Identity** for authentication (no API keys)
- **Private Endpoints** for network isolation
- Infrastructure is defined in **Bicep** (`enterprise/infra/`)

### 5. Container Build

Always target `linux/amd64` when building Docker images — Foundry runs on AMD64:

```bash
docker build --platform linux/amd64 -t my-agent:latest .
```

### 6. Python Dependencies

Use `uv` (fast pip alternative) for dependency installation inside Docker. See `Dockerfile` for the pattern. Key packages:

- `azure-ai-agentserver-agentframework` — hosting adapter
- `agent-framework-core` — core agent framework
- `agent-framework-azure-ai` — Azure AI integration
- `azure-ai-projects` — Foundry deployment SDK
- `azure-identity` — authentication

### 7. Coding Style

- Keep `main.py` focused — it's the ONLY file that matters for containerization
- Use heavy comments explaining what's required vs. optional
- Configuration via `os.getenv()` — no hardcoded values
- Trace with OpenTelemetry when `APPLICATIONINSIGHTS_CONNECTION_STRING` is set
