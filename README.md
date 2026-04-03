# kagent

Deploy AI agents on Kubernetes — backed by Helm, powered by MCP, routed through AgentGateway.

> **Status:** Single-agent setup is functional. Multi-agent is work in progress.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Kubernetes cluster | kind / EKS / AKS / GKE |
| `kubectl` | latest stable |
| `helm` | >= 3 |
| ArgoCD | installed in cluster |
| LLM API key | OpenAI / Gemini / Groq |

---

## Installation

### 1. Install kagent CRDs

CRDs must be installed before the controller.
```bash
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --namespace kagent \
  --create-namespace
```

Registers: `Agent`, `ModelConfig`, `ToolServer`, `MCPIntegration`

---

### 2. Install kagent

Export your API key:
```bash
export OPENAI_API_KEY="your-api-key"
```

Install:
```bash
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --namespace kagent \
  --set providers.openAI.apiKey=$OPENAI_API_KEY
```

**Optional** — access the UI:
```bash
kubectl port-forward svc/kagent-ui -n kagent 8080:8080
```

Open `http://localhost:8080`

---

### 3. Install AgentGateway

Install Kubernetes Gateway API CRDs:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml
```

Install AgentGateway CRDs and controller:
```bash
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v1.0.1 agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds

helm upgrade -i -n agentgateway-system agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
  --version v1.0.1
```

---

## Deploy the Crossplane Agent

With ArgoCD in place, apply the App of Apps:
```bash
kubectl apply -f argocd/apps/apps-of-apps.yaml
```

This deploys:
- AgentGateway with the Crossplane MCP server as a backend
- The Crossplane MCP server itself
- kagent with an agent configured to consume it via `RemoteMCPServer`

---

## Repository Structure
```
.
├── agentgateway/            # Gateway, HTTPRoute, CORS, and policy configs
├── argocd/                  # App of Apps — deploys everything via GitOps
├── mcp-server/              # Crossplane MCP server deployment and RBAC
├── single-agent/            # Single-agent setup (functional)
├── multi-agent/             # Multi-agent orchestration (not tested)
├── postgres/                # Database for agent registry state
└── skills/                  # Custom MCP tool definitions
```

### Key files
```
argocd/
├── apps/apps-of-apps.yaml       # Entry point — apply this first
├── agentgateway-app.yaml
├── crossplane-app.yaml
├── crossplane-mcp-app.yaml
└── kagent-app.yaml

single-agent/
├── crossplane-agent.yaml        # kagent Agent definition
├── crossplane-mcp-server.yaml   # RemoteMCPServer config
├── agentgateway-crossplane-mcp.yaml
└── groq-modelconfig.yaml        # Example ModelConfig for Groq

agentgateway/
├── gateway.yaml
├── httproute.yaml
├── mcp-backend.yaml
├── crossplane-mcp-cors.yaml
└── agentgateway_policies.yaml
```

---

## Multi-agent

Multi-agent manifests are present under `multi-agent/` but are **not tested**. Contributions welcome.