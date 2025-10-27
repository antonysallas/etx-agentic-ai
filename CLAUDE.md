# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the ETX Agentic AI Hackathon repository, demonstrating a comprehensive agentic AI use case built on LLaMA Stack (LLS) with Model Context Protocol (MCP) integration. The project showcases enterprise-grade AI agent deployment patterns on OpenShift/Kubernetes using GitOps principles.

**Core Components:**
- Python-based AI agents using LLaMA Stack Client for autonomous task execution
- MCP servers for GitHub, OpenShift, and external service integration
- Built-in tools for internet search (Tavily/Brave)
- FastAPI service for pipeline failure reporting and automated issue creation
- Complete Kubernetes/OpenShift infrastructure as code
- Observability stack with OpenTelemetry integration

## Development Commands

### Python Environment
```bash
# Python 3.12+ required (see .python-version)
# Dependencies managed via uv (see uv.lock and pyproject.toml)
python -m venv .venv
source .venv/bin/activate
pip install llama-stack-client==0.2.15
```

### Running the Agent Service
```bash
# Main FastAPI service (code/main.py)
cd code
uvicorn main:app --host 0.0.0.0 --port 8000

# Direct agent execution (code/agent.py)
python code/agent.py
```

### Port Forwarding to LLaMA Stack
```bash
# Forward LLaMA Stack service (default port 8321)
make port-forward PORT=8321 NS_LLAMA=llama-stack
```

### OpenShift/Kubernetes Operations
```bash
# Trigger a pipeline run (requires GITHUB_USER)
make trigger-bad-run GITHUB_USER=<your-username> NS_AGENT=ai-agent

# Apply infrastructure using kustomize
kustomize build infra/applications/llama-stack/base | oc apply -f-

# Deploy QuickStarts
oc apply -k infra/quickstarts/
```

### Helm Chart Generation
```bash
# Generate tenant manifests from Helm chart
cd code/chart
helm template ai-agent . --values values.yaml --output-dir ../../../infra/tenants/ai-agent/base/sno

# Move templates to correct location
cd ../../../infra/tenants/ai-agent/base/sno
mv ai-agent/templates/* templates/
rm -rf ai-agent/
```

### Documentation (Antora)
```bash
# Run documentation server with Podman (recommended)
podman run --rm --name antora -v $PWD:/antora -p 8080:8080 -i -t ghcr.io/juliaaano/antora-viewer

# For SELinux environments
podman run --rm --name antora -v $PWD:/antora:z -p 8080:8080 -i -t ghcr.io/juliaaano/antora-viewer

# Local build (alternative)
./showroom-utils/lab-serve  # Start server on http://localhost:8080
./showroom-utils/lab-build  # Build static HTML
```

## Architecture

### Agent Service Architecture
The system follows an event-driven architecture where pipeline failures trigger autonomous agent analysis:

1. **FastAPI Service** (`code/main.py`): Exposes `/report-failure` endpoint accepting PipelineFailure payloads
2. **Agent Logic** (`code/agent.py`): ReActAgent implementation that:
   - Retrieves OpenShift pod logs using MCP tools
   - Analyzes errors using LLM inference
   - Searches for solutions via web search tools
   - Creates GitHub issues with structured error reports
3. **Utils** (`code/utils.py`): Provides `step_printer()` for formatted agent step visualization

### LLaMA Stack Configuration
LLaMA Stack is deployed as a custom resource (`LlamaStackDistribution`) with:
- **Distribution**: `remote-vllm` (image: `quay.io/eformat/distribution-remote-vllm:0.2.15`)
- **Models**:
  - `granite-31-2b-instruct` (local vLLM)
  - `llama-3-2-3b` (MAAS remote)
  - `llama-4-scout-17b-16e-w4a16` (MAAS remote)
  - `all-MiniLM-L6-v2` (embeddings)
- **Tools**: MCP servers (GitHub, OpenShift), Tavily Search
- **Telemetry**: OpenTelemetry export to collector at `otel-collector-collector.observability-hub.svc.cluster.local:4318`

Configuration is stored in ConfigMap `llama-stack-config` (see `infra/applications/llama-stack/base/configmap.yaml`).

### Infrastructure Organization
The `infra/` directory follows Kustomize overlay patterns:

```
infra/
├── applications/          # Application-specific resources
│   ├── llama-stack/       # LlamaStackDistribution, ConfigMap, Secrets
│   ├── mcp-*/             # MCP server deployments (github, openshift, weather)
│   ├── observability/     # OTel collector, Tempo, Grafana
│   ├── gpu/               # GPU resource management
│   ├── models/            # Model serving configurations
│   └── vault/             # Secret management
├── bootstrap/             # Initial cluster setup (ArgoCD, ACM, auth, certs, storage)
├── app-of-apps/           # GitOps app-of-apps pattern
├── tenants/               # Tenant workload manifests (generated from Helm)
└── quickstarts/           # OpenShift QuickStart resources
```

Each application follows `base/` + `overlay/<env>/` structure for environment-specific customization.

### Agent Environment Variables
Key environment variables used by `code/agent.py`:
- `LLAMA_STACK_URL`: LLaMA Stack endpoint (default: `http://llamastack-with-config-service.llama-stack.svc.cluster.local:8321`)
- `MODEL_ID`: Model to use (default: `granite-31-2b-instruct`)
- `TEMPERATURE`: Sampling temperature (default: `0.0`)
- `MAX_TOKENS`: Maximum response tokens (default: `512`)
- `CLIENT_TIMEOUT`: Client timeout in seconds (default: `600.0`)
- `MAX_INFER_ITERATIONS`: Max ReAct iterations (default: `10`)

### MCP Integration
MCP (Model Context Protocol) servers enable tool-based interactions:
- **mcp::openshift**: Query pod logs, cluster resources
- **mcp::github**: Create issues, PRs, manage repositories
- **builtin::websearch**: Tavily/Brave search for solution lookup

MCP servers are deployed as separate pods in the infrastructure and registered with LLaMA Stack via the ConfigMap.

### Agent Prompting Strategy
The agent in `code/agent.py` uses few-shot prompting with structured examples (lines 77-102) to ensure:
- Consistent error categorization
- Structured GitHub issue body format with sections: Cluster/namespace location, Summary, Detailed error, Possible solutions
- JSON-formatted tool calls for `create_issue`

The prompt explicitly instructs to tail only last 10 lines of pod logs to reduce token usage.

## Key Implementation Details

### ReActAgent Configuration
```python
agent = ReActAgent(
    client=client,
    model=model_id,
    tools=["mcp::openshift", "builtin::websearch", "mcp::github"],
    response_format={
        "type": "json_schema",
        "json_schema": ReActOutput.model_json_schema(),
    },
    sampling_params={"max_tokens":max_tokens},
    max_infer_iters=max_infer_iterations
)
```

The agent enforces JSON schema responses and iterates up to `max_infer_iterations` to complete tool-calling sequences.

### Container Build
Application containerization uses `code/Containerfile` with `code/requirements.txt` for dependencies. The image is built via Tekton pipelines (see `infra/applications` for pipeline definitions).

### Observability
LLaMA Stack exports telemetry to multiple sinks:
- `console`: Stdout logging
- `sqlite`: Local persistence
- `otel_trace`: Distributed tracing via OpenTelemetry
- `otel_metric`: Metrics via OpenTelemetry

Traces/metrics flow to the observability hub namespace for visualization in Grafana/Tempo.

## Working with this Repository

### Adding New MCP Servers
1. Create new directory under `infra/applications/mcp-<name>/`
2. Add base Kubernetes resources (Deployment, Service, ConfigMap)
3. Register in LLaMA Stack ConfigMap under `providers.tool_runtime`
4. Update agent's `tools` list in `code/agent.py`

### Modifying Agent Behavior
- **Prompt engineering**: Edit the prompt template in `code/agent.py` (lines 77-102)
- **Tool selection**: Modify `tools` parameter in ReActAgent instantiation
- **Model parameters**: Adjust `temperature`, `max_tokens`, `max_infer_iterations` via environment variables
- **Response format**: The agent uses strict JSON schema validation for tool calls

### Deploying Infrastructure Changes
The repository uses GitOps principles:
1. Modify resources in `infra/applications/<app>/base/` or overlays
2. Commit changes to Git
3. ArgoCD syncs changes automatically (if using bootstrap setup)
4. For manual testing: `kustomize build <path> | oc apply -f-`

### Testing Agents Locally
```bash
# Set environment variables
export LLAMA_STACK_URL=http://localhost:8321
export MODEL_ID=granite-31-2b-instruct
export TAVILY_API_KEY=<your-key>

# Port forward LLaMA Stack
make port-forward

# Run agent standalone
python code/agent.py

# Or run FastAPI service
cd code && uvicorn main:app --reload
```

### Jupyter Notebooks
- `notebooks/agent-prototyping.ipynb`: Agent experimentation and testing
- `code/getting-started.ipynb`: Quick start guide
- `code/test_client.ipynb`, `code/test_server.ipynb`: Client/server testing

Set up notebook environment using `notebooks/requirements.txt` and `notebooks/env` for environment variables.

## Additional Context

### Bootstrap Process
The cluster setup follows a specific order (see `infra/README.md`):
1. Install ArgoCD and ACM: `kustomize build --enable-helm bootstrap | oc apply -f-`
2. Create CRs: `oc apply -f gitops/bootstrap/setup-cr.yaml`
3. Configure auth, PKI, storage using bootstrap scripts
4. Install app-of-apps: `oc apply -f gitops/app-of-apps/sno-app-of-apps.yaml`
5. Setup Vault for secrets management

### Documentation Content
Educational content is in `content/modules/ROOT/`:
- `pages/`: Lab modules (AsciiDoc format)
- `assets/images/`: Diagrams and screenshots
- `examples/`: Downloadable scripts and assets
- `nav.adoc`: Lab navigation structure

### Alternative Agent Implementation
The repository includes a DSPy-based agent example (`code/dspy-mcp-agent4.py`, `code/dspy-lls-tool-test.py`) demonstrating context engineering patterns as an alternative to the main ReAct agent.

### Evaluation Framework
The `evals/` directory contains evaluation scripts for testing agent performance and accuracy.
