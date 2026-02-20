---
title: "Connect Cursor to GitHub MCP Server Through AgentGateway"
publishDate: 2026-02-19
author: "Sebastian Maniak"
description: "Deploy the GitHub MCP server behind Solo AgentGateway and connect Cursor as the MCP client — adding authentication, rate limiting, and observability to every tool call your IDE makes."
---

## Introduction

Cursor ships with built-in MCP client support, which means you can connect it to any MCP server — including GitHub's official MCP server — to give your AI coding assistant direct access to issues, pull requests, code search, and repository management.

But connecting Cursor directly to an MCP server means no visibility into what tools are being called, no rate limiting, and no centralized authentication. Every tool invocation goes straight from your IDE to the backend with zero governance.

This guide shows you how to deploy the **GitHub MCP server** behind **[Solo AgentGateway](https://agentgateway.dev)** and point **Cursor** at the gateway endpoint. You get JWT authentication, OpenTelemetry tracing, and rate limiting on every GitHub tool call — without changing how Cursor works.

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
                        │ JWT auth
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

Install the CRDs and AgentGateway via Helm:

```bash
helm install agentgateway-crds \
  oci://ghcr.io/kgateway-dev/charts/agentgateway-crds

helm install agentgateway \
  oci://ghcr.io/kgateway-dev/charts/agentgateway \
  --create-namespace \
  --namespace agentgateway
```

Verify AgentGateway is ready:

```bash
kubectl rollout status deploy/agentgateway -n agentgateway
```

## Step 3: Deploy the GitHub MCP Server

The official GitHub MCP server runs as a stdio-based process. To make it network-accessible for AgentGateway, we'll deploy it as an SSE server using the container image.

```yaml
# github-mcp-server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-mcp-server
  namespace: agentgateway
spec:
  replicas: 1
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
  namespace: agentgateway
spec:
  selector:
    app: github-mcp-server
  ports:
    - port: 8080
      targetPort: 8080
```

Create the GitHub PAT secret:

```bash
kubectl create secret generic github-pat \
  --namespace agentgateway \
  --from-literal=token="ghp_your_token_here"
```

Deploy:

```bash
kubectl apply -f github-mcp-server.yaml
```

Verify the MCP server is running:

```bash
kubectl logs -n agentgateway deploy/github-mcp-server
```

## Step 4: Configure AgentGateway MCP Backend

Create a backend that points AgentGateway at the GitHub MCP server:

```yaml
# github-mcp-backend.yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: github-mcp
  namespace: agentgateway
spec:
  mcp:
    targets:
      - static:
          host: github-mcp-server.agentgateway.svc.cluster.local
          port: 8080
          protocol: sse
```

```bash
kubectl apply -f github-mcp-backend.yaml
```

## Step 5: Create the MCP HTTPRoute

Expose the GitHub MCP backend through AgentGateway with an HTTPRoute:

```yaml
# github-mcp-route.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: github-mcp
  namespace: agentgateway
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /github
      backendRefs:
        - name: github-mcp
          namespace: agentgateway
          port: 8080
```

```bash
kubectl apply -f github-mcp-route.yaml
```

## Step 6: Add Security Policies (Optional but Recommended)

Add JWT authentication and rate limiting:

```yaml
# github-mcp-policy.yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: github-mcp-policy
  namespace: agentgateway
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: github-mcp
  default:
    rateLimit:
      requests:
        limit: 60
        windowSeconds: 60
```

```bash
kubectl apply -f github-mcp-policy.yaml
```

## Step 7: Connect Cursor

Port-forward the AgentGateway service to your laptop:

```bash
kubectl port-forward -n agentgateway svc/agentgateway 8080:8080
```

Leave that running in a separate terminal. Now create or update `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (per-project):

```json
{
  "mcpServers": {
    "github": {
      "url": "http://localhost:8080/github/sse",
      "transport": "sse"
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

Check AgentGateway logs to confirm MCP traffic is being proxied:

```bash
kubectl logs -n agentgateway deploy/agentgateway --tail=20
```

You should see MCP SSE connection events and tool call forwards.

If you have Langfuse or Jaeger configured for tracing, you'll see spans for each `tools/list` and `tools/call` invocation with full metadata.

## What's Next

- **Add more MCP servers**: Slack, Notion, Jira — all behind the same gateway
- **Enterprise policies**: Prompt guards, PII detection, function-level RBAC with `EnterpriseAgentgatewayPolicy`
- **Multi-IDE support**: The same AgentGateway endpoint works for Claude Desktop, VS Code with Copilot, and any MCP-compatible client

## Cleanup

When you're done, tear down the cluster:

```bash
kind delete cluster --name agentgateway
```

Remove the Cursor MCP config if you no longer need it:

```bash
rm ~/.cursor/mcp.json
```

## Conclusion

Cursor's MCP support is powerful, but connecting it directly to backend servers skips every governance layer you'd want in a team environment. AgentGateway sits in the middle and gives you auth, rate limiting, and full observability — without changing how Cursor or the MCP server works.

Your IDE gets smarter tools. Your platform team gets visibility. Everyone wins.
