---
title: "Connect Cursor to GitHub MCP Server Through AgentGateway"
publishDate: 2026-02-19
author: "Sebastian Maniak"
description: "Route Cursor's MCP traffic to GitHub's remote MCP server through Solo AgentGateway — adding authentication, rate limiting, and observability to every tool call your IDE makes."
---

## Introduction

Cursor ships with built-in MCP client support, which means you can connect it to any MCP server — including GitHub's official remote MCP server — to give your AI coding assistant direct access to issues, pull requests, code search, and repository management.

But connecting Cursor directly to GitHub's MCP server means no visibility into what tools are being called, no rate limiting, and no centralized authentication. Every tool invocation goes straight from your IDE to GitHub with zero governance.

This guide shows you how to route Cursor's MCP traffic to **GitHub's remote MCP server** through **[Solo AgentGateway](https://agentgateway.dev)**. You get authentication, OpenTelemetry tracing, and rate limiting on every GitHub tool call — without changing how Cursor works.

## What You Get

- **Proxied MCP traffic**: Cursor talks to AgentGateway, which forwards to GitHub's remote MCP server
- **Authentication**: Centralized GitHub PAT management — tokens live in Kubernetes secrets, not on developer laptops
- **Rate Limiting**: Control how many tool calls per minute your IDE can make
- **Observability**: OpenTelemetry traces for every `list_tools`, `call_tool`, and resource request
- **No self-hosted MCP server**: GitHub hosts the MCP server at `api.githubcopilot.com` — you just proxy to it

## Architecture

```
┌────────────┐        ┌──────────────────┐        ┌──────────────────────────┐
│   Cursor    │──MCP──▶│  AgentGateway    │──MCP──▶│  api.githubcopilot.com   │
│  (IDE)      │        │  (proxy)         │        │  (GitHub Remote MCP)     │
└────────────┘        └──────────────────┘        └──────────────────────────┘
                        │ Auth
                        │ Rate limiting
                        │ OTLP traces
```

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed
- `kubectl` and `helm` installed
- A [GitHub Personal Access Token](https://github.com/settings/tokens) (PAT)
- [Cursor](https://cursor.com) installed locally

## Step 1: Create a Kind Cluster

```bash
kind create cluster --name agentgateway
```

Verify it's running:

```bash
kubectl get nodes
```

## Step 2: Install AgentGateway

Deploy the Kubernetes Gateway API CRDs:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

Install the AgentGateway CRDs and control plane:

```bash
helm upgrade -i agentgateway-crds \
  oci://ghcr.io/kgateway-dev/charts/agentgateway-crds

helm upgrade -i agentgateway \
  oci://ghcr.io/kgateway-dev/charts/agentgateway \
  -n agentgateway-system --create-namespace
```

Verify the control plane is running:

```bash
kubectl get pods -n agentgateway-system
```

## Step 3: Create an AgentGateway Proxy

Create a Gateway resource that spins up an agentgateway proxy:

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - name: http
    port: 80
    protocol: HTTP
EOF
```

Wait for the proxy pod to be ready:

```bash
kubectl rollout status deploy/agentgateway-proxy -n agentgateway-system
```

## Step 4: Configure the GitHub Remote MCP Backend

GitHub hosts a remote MCP server at `https://api.githubcopilot.com/mcp/` that supports streamable HTTP. Instead of deploying our own GitHub MCP server, we'll point AgentGateway at GitHub's hosted endpoint.

Create a secret with your GitHub PAT for backend authentication:

```bash
kubectl create secret generic github-pat \
  -n agentgateway-system \
  --from-literal=Authorization="Bearer ghp_your_token_here"
```

Create an `AgentgatewayBackend` that targets GitHub's remote MCP server:

```bash
kubectl apply -f- <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: github-mcp
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: github
      static:
        host: api.githubcopilot.com
        port: 443
        protocol: StreamableHTTP
  policies:
    auth:
      secretRef:
        name: github-pat
        namespace: agentgateway-system
EOF
```

## Step 5: Create the MCP HTTPRoute

Route traffic from the AgentGateway proxy to the GitHub MCP backend:

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: github-mcp
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
  - backendRefs:
    - name: github-mcp
      group: agentgateway.dev
      kind: AgentgatewayBackend
EOF
```

## Step 6: Connect Cursor

Port-forward the AgentGateway proxy to your laptop:

```bash
kubectl port-forward -n agentgateway-system deployment/agentgateway-proxy 8080:80
```

Leave that running in a separate terminal. Now create or update `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (per-project):

```json
{
  "mcpServers": {
    "github": {
      "url": "http://localhost:8080/mcp",
      "transport": "streamable-http"
    }
  }
}
```

Restart Cursor. Open **Cursor Settings → MCP** and verify the GitHub server shows as connected with its tools listed.

## Step 7: Test It

Open Cursor's AI chat and try:

- *"List open issues in my repo"*
- *"Create a new branch called feature/test"*
- *"Search for files referencing AgentGateway"*

Each of these triggers MCP tool calls that flow through AgentGateway to GitHub's remote MCP server. You'll see them in your OTLP traces if you have observability configured.

## Verifying Traffic Flows Through the Gateway

Check AgentGateway proxy logs to confirm MCP traffic is being proxied:

```bash
kubectl logs -n agentgateway-system deploy/agentgateway-proxy --tail=20
```

You should see MCP connection events and tool call forwards to `api.githubcopilot.com`.

## What's Next

- **Add more MCP servers**: Slack, Notion, Jira — all behind the same gateway
- **Security policies**: JWT auth, prompt guards, rate limiting with `AgentgatewayPolicy`
- **Multi-IDE support**: The same AgentGateway endpoint works for Claude Desktop, VS Code with Copilot, and any MCP-compatible client

## Cleanup

```bash
kubectl delete httproute github-mcp -n agentgateway-system
kubectl delete agentgatewaybackend github-mcp -n agentgateway-system
kubectl delete secret github-pat -n agentgateway-system
kind delete cluster --name agentgateway
```

## Conclusion

Cursor's MCP support is powerful, but connecting it directly to GitHub skips every governance layer you'd want in a team environment. AgentGateway sits in the middle and gives you auth, rate limiting, and full observability — without deploying any MCP servers yourself. GitHub hosts the MCP server, AgentGateway proxies and governs the traffic, and Cursor just works.

Your IDE gets smarter tools. Your platform team gets visibility. Everyone wins.
