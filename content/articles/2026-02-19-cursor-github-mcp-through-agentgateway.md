---
title: "Connect Cursor to GitHub MCP Server Through AgentGateway"
publishDate: 2026-02-19
author: "Sebastian Maniak"
description: "Deploy the GitHub MCP server behind Solo AgentGateway and connect Cursor as the MCP client — adding authentication, rate limiting, and observability to every tool call your IDE makes."
---

## Introduction

Cursor ships with built-in MCP client support, which means you can connect it to any MCP server — including GitHub's official MCP server — to give your AI coding assistant direct access to issues, pull requests, code search, and repository management.

But connecting Cursor directly to an MCP server means no visibility into what tools are being called, no rate limiting, and no centralized authentication. Every tool invocation goes straight from your IDE to the backend with zero governance.

This guide shows you how to deploy the **GitHub MCP server** behind **[Solo AgentGateway](https://agentgateway.dev)** and point **Cursor** at the gateway endpoint. You get authentication, OpenTelemetry tracing, and rate limiting on every GitHub tool call — without changing how Cursor works.

## What You Get

- **Proxied MCP traffic**: Cursor talks to AgentGateway, which forwards to the GitHub MCP server
- **Authentication**: JWT validation on every MCP request before it hits your backend
- **Rate Limiting**: Control how many tool calls per minute your IDE can make
- **Observability**: OpenTelemetry traces for every `list_tools`, `call_tool`, and resource request
- **Centralized Secrets**: Your GitHub PAT lives in a Kubernetes secret, not in Cursor's config

## Architecture

```
┌────────────┐        ┌──────────────────┐        ┌──────────────────┐
│   Cursor    │──MCP──▶│  AgentGateway    │──MCP──▶│  GitHub MCP      │
│  (IDE)      │  SSE   │  (proxy)         │        │  Server          │
└────────────┘        └──────────────────┘        └──────────────────┘
                        │ Auth
                        │ Rate limiting
                        │ OTLP traces
```

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed
- `kubectl` and `helm` installed
- A [GitHub Personal Access Token](https://github.com/settings/tokens) (PAT) with `repo` scope
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

## Step 4: Deploy the GitHub MCP Server

The official [GitHub MCP server](https://github.com/github/github-mcp-server) provides tools for issues, PRs, code search, and repository management. We'll deploy it as an SSE server so AgentGateway can proxy to it.

Create the GitHub PAT secret:

```bash
kubectl create secret generic github-pat \
  --from-literal=token="ghp_your_token_here"
```

Deploy the server:

```bash
kubectl apply -f- <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-mcp-server
spec:
  selector:
    matchLabels:
      app: github-mcp-server
  template:
    metadata:
      labels:
        app: github-mcp-server
    spec:
      containers:
      - name: github-mcp
        image: ghcr.io/github/github-mcp-server:latest
        args: ["--transport", "sse", "--host", "0.0.0.0", "--port", "8080"]
        ports:
        - containerPort: 8080
        env:
        - name: GITHUB_PERSONAL_ACCESS_TOKEN
          valueFrom:
            secretKeyRef:
              name: github-pat
              key: token
---
apiVersion: v1
kind: Service
metadata:
  name: github-mcp-server
  labels:
    app: github-mcp-server
spec:
  selector:
    app: github-mcp-server
  ports:
  - port: 8080
    targetPort: 8080
    appProtocol: kgateway.dev/mcp
EOF
```

> **Important**: The `appProtocol: kgateway.dev/mcp` annotation on the Service tells AgentGateway to use the MCP protocol when connecting to this backend.

Verify the MCP server is running:

```bash
kubectl logs deploy/github-mcp-server
```

## Step 5: Configure AgentGateway MCP Backend

Create an `AgentgatewayBackend` that points AgentGateway at the GitHub MCP server:

```bash
kubectl apply -f- <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: github-mcp
spec:
  mcp:
    targets:
    - name: github
      static:
        host: github-mcp-server.default.svc.cluster.local
        port: 8080
        protocol: SSE
EOF
```

## Step 6: Create the MCP HTTPRoute

Route traffic from the AgentGateway proxy to the GitHub MCP backend:

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: github-mcp
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

## Step 7: Connect Cursor

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

## Step 8: Test It

Open Cursor's AI chat and try:

- *"List open issues in my repo"*
- *"Create a new branch called feature/test"*
- *"Search for files referencing AgentGateway"*

Each of these triggers MCP tool calls that flow through AgentGateway. You'll see them in your OTLP traces if you have observability configured.

## Verifying Traffic Flows Through the Gateway

Check AgentGateway proxy logs to confirm MCP traffic is being proxied:

```bash
kubectl logs -n agentgateway-system deploy/agentgateway-proxy --tail=20
```

You should see MCP connection events and tool call forwards.

If you have Langfuse or Jaeger configured for tracing, you'll see spans for each `tools/list` and `tools/call` invocation with full metadata.

## What's Next

- **Add more MCP servers**: Slack, Notion, Jira — all behind the same gateway
- **Security policies**: JWT auth, prompt guards, rate limiting with `AgentgatewayPolicy`
- **Multi-IDE support**: The same AgentGateway endpoint works for Claude Desktop, VS Code with Copilot, and any MCP-compatible client

## Cleanup

```bash
kubectl delete httproute github-mcp
kubectl delete agentgatewaybackend github-mcp
kubectl delete deploy github-mcp-server
kubectl delete svc github-mcp-server
kubectl delete secret github-pat
kind delete cluster --name agentgateway
```

## Conclusion

Cursor's MCP support is powerful, but connecting it directly to backend servers skips every governance layer you'd want in a team environment. AgentGateway sits in the middle and gives you auth, rate limiting, and full observability — without changing how Cursor or the MCP server works.

Your IDE gets smarter tools. Your platform team gets visibility. Everyone wins.
