---
title: "MCP Multiplexing and Tool Access Control with AgentGateway"
publishDate: 2026-02-20
author: "Sebastian Maniak"
description: "Federate multiple MCP servers behind a single endpoint with AgentGateway OSS — then lock down tool access with JWT-based RBAC so agents only see the tools they're allowed to use."
---

## Introduction

As your agentic AI environment grows, you end up with a sprawl of MCP servers — one for time utilities, one for general-purpose tools, one for Slack, another for GitHub. Each server exposes different tools, runs on a different port, and needs its own connection configuration.

Your MCP clients (Cursor, Claude Desktop, VS Code) shouldn't need to know about every server individually. They should connect to **one endpoint** and see **all available tools** — while you control exactly **which tools each user or agent can access**.

This guide walks you through two powerful AgentGateway features:

1. **MCP Multiplexing** — federate multiple MCP servers behind a single gateway endpoint
2. **Tool Access Control** — use JWT-based RBAC to restrict which tools are visible per user

We'll deploy two MCP servers (`mcp-server-everything` for utility tools and `mcp-website-fetcher` for fetching web content), multiplex them behind AgentGateway, connect from any IDE, and then lock down tool access so only authorized users can see specific tools.

## What You Get

- **One endpoint, many servers**: Clients connect to `/mcp` and see tools from all federated servers
- **Automatic tool namespacing**: Tools are prefixed with the server name to avoid conflicts
- **Label-based federation**: Add servers by just adding a Kubernetes label — no config changes
- **JWT-based tool RBAC**: Control tool visibility per user based on JWT claims
- **IDE-agnostic**: Works with Cursor, VS Code, Claude Code, Windsurf, OpenCode

## Architecture

```
┌──────────────┐
│   Cursor     │──┐
├──────────────┤  │
│   VS Code    │──┤
├──────────────┤  │       ┌──────────────────┐       ┌──────────────────────┐
│  Claude Code │──┼─MCP──▶│  AgentGateway    │──────▶│  mcp-server-everything│
├──────────────┤  │       │  /mcp endpoint   │       │  (echo, add, etc.)   │
│   Windsurf   │──┘       │                  │──────▶├──────────────────────┤
└──────────────┘          │  Multiplexing    │       │  mcp-website-fetcher │
                          │  + Tool RBAC     │       │  (fetch web content) │
                          └──────────────────┘       └──────────────────────┘
                            │ JWT validation
                            │ CEL-based tool filtering
                            │ OTLP traces
```

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed
- `kubectl` and `helm` installed
- At least one MCP client: Cursor, VS Code, Claude Code, Windsurf, or OpenCode

## Step 1: Create a Kind Cluster

```bash
kind create cluster --name agentgateway-mcp
```

Verify it's running:

```bash
kubectl get nodes
```

## Step 2: Install AgentGateway OSS

Deploy the Kubernetes Gateway API CRDs:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

Install the AgentGateway CRDs and control plane:

```bash
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v2.2.1 \
  agentgateway-crds oci://ghcr.io/kgateway-dev/charts/agentgateway-crds

helm upgrade -i -n agentgateway-system \
  agentgateway oci://ghcr.io/kgateway-dev/charts/agentgateway \
  --version v2.2.1
```

Verify the control plane is running:

```bash
kubectl get pods -n agentgateway-system
```

You should see the `agentgateway` control plane pod in `Running` state.

## Step 3: Create an AgentGateway Proxy

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: mcp-gateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - name: http
    port: 80
    protocol: HTTP
EOF
```

Wait for the proxy pod:

```bash
kubectl rollout status deploy/mcp-gateway -n agentgateway-system --timeout=60s
```

## Step 4: Deploy Two MCP Servers

### Server 1: mcp-server-everything

This server provides general utility tools like `echo`, `add`, `longRunningOperation`, and more.

```bash
kubectl apply -f- <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-server-everything
  namespace: agentgateway-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-server-everything
  template:
    metadata:
      labels:
        app: mcp-server-everything
    spec:
      containers:
      - name: mcp-server-everything
        image: docker.io/mcp/everything:latest
        ports:
        - containerPort: 3001
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-server-everything
  namespace: agentgateway-system
  labels:
    app: mcp-server-everything
    mcp-federation: "true"
spec:
  ports:
  - port: 3001
    targetPort: 3001
    appProtocol: kgateway.dev/mcp
  selector:
    app: mcp-server-everything
EOF
```

### Server 2: mcp-website-fetcher

This server provides a `fetch` tool that can retrieve and extract content from any URL — useful for giving your AI assistant web browsing capabilities.

```bash
kubectl apply -f- <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-website-fetcher
  namespace: agentgateway-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-website-fetcher
  template:
    metadata:
      labels:
        app: mcp-website-fetcher
    spec:
      containers:
      - name: mcp-website-fetcher
        image: docker.io/mcp/fetch:latest
        ports:
        - containerPort: 3001
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-website-fetcher
  namespace: agentgateway-system
  labels:
    app: mcp-website-fetcher
    mcp-federation: "true"
spec:
  ports:
  - port: 3001
    targetPort: 3001
    appProtocol: kgateway.dev/mcp
  selector:
    app: mcp-website-fetcher
EOF
```

Wait for both servers to be ready:

```bash
kubectl rollout status deploy/mcp-server-everything -n agentgateway-system --timeout=60s
kubectl rollout status deploy/mcp-website-fetcher -n agentgateway-system --timeout=60s
```

> **Key detail**: Both services have the label `mcp-federation: "true"` and `appProtocol: kgateway.dev/mcp`. This is how AgentGateway discovers and federates them.

## Step 5: Create the Multiplexed MCP Backend

Here's where the magic happens. Instead of pointing to individual servers, you use a **label selector** to match all services with `mcp-federation: "true"`:

```bash
kubectl apply -f- <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mcp-federated
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: mcp-servers
      selector:
        services:
          matchLabels:
            mcp-federation: "true"
EOF
```

This single backend automatically discovers both `mcp-server-everything` and `mcp-website-fetcher` — and any future MCP server you deploy with the `mcp-federation: "true"` label.

## Step 6: Create the HTTPRoute

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp-route
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: mcp-gateway
    namespace: agentgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mcp
    backendRefs:
    - name: mcp-federated
      group: agentgateway.dev
      kind: AgentgatewayBackend
EOF
```

## Step 7: Port-Forward and Test

Expose the gateway locally:

```bash
kubectl port-forward -n agentgateway-system deployment/mcp-gateway 8080:80
```

### Test with MCP Inspector

```bash
npx @modelcontextprotocol/inspector@latest
```

Connect with:
- **Transport**: Streamable HTTP
- **URL**: `http://localhost:8080/mcp`

Click **Tools** → **List Tools**. You should see tools from **both** servers, namespaced:

- `mcp-server-everything-3001_echo` — echo a message
- `mcp-server-everything-3001_add` — add two numbers
- `mcp-server-everything-3001_longRunningOperation` — simulate a long task
- `mcp-website-fetcher_fetch` — fetch and extract content from a URL

**One endpoint. Two servers. All tools.**

### Test with Curl

You can also verify the MCP connection with curl:

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc": "2.0", "method": "tools/list", "id": 1}'
```

You should see a JSON response listing all tools from both servers.

---

## Step 8: Connect from Your IDE

With port-forward running, configure your IDE:

### Cursor

Create or edit `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "federated-tools": {
      "url": "http://localhost:8080/mcp"
    }
  }
}
```

### VS Code (with GitHub Copilot)

Add to `settings.json`:

```json
{
  "github.copilot.chat.mcp.servers": {
    "federated-tools": {
      "url": "http://localhost:8080/mcp"
    }
  }
}
```

### Claude Code

```bash
claude mcp add federated-tools --transport sse http://localhost:8080/mcp
```

### Windsurf

Create or edit `~/.windsurf/mcp.json`:

```json
{
  "mcpServers": {
    "federated-tools": {
      "url": "http://localhost:8080/mcp"
    }
  }
}
```

### OpenCode

Add to `opencode.json`:

```json
{
  "mcp": {
    "servers": {
      "federated-tools": {
        "type": "sse",
        "url": "http://localhost:8080/mcp"
      }
    }
  }
}
```

Now try asking your IDE:

- *"Echo back: AgentGateway is awesome"* — uses the everything server
- *"Add 42 and 58"* — uses the everything server
- *"Fetch the content from https://agentgateway.dev"* — uses the website fetcher

The IDE doesn't know or care that these tools come from different servers. It's all one MCP endpoint.

---

## Step 9: Add Tool Access Control with JWT RBAC

Now let's lock things down. We'll configure AgentGateway to validate JWT tokens and restrict which tools each user can see.

### Generate JWT Tokens

We'll use pre-generated tokens for two users. You can generate your own with the [JWT generator tool](https://github.com/kgateway-dev/kgateway/blob/main/hack/utils/jwt/jwt-generator.go).

Save the JWT for Alice (dev team):

```bash
export ALICE_JWT="eyJhbGciOiJSUzI1NiIsImtpZCI6IjU4OTE2NDUwMzIxNTk4OTQzODMiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJzb2xvLmlvIiwic3ViIjoiYWxpY2UiLCJ0ZWFtIjoiZGV2IiwiZXhwIjoyMDczNjcwNDgyLCJuYmYiOjE3NjYwODY0ODIsImlhdCI6MTc2NjA4NjQ4Mn0.C-KYZsfWwlwRw4cKHXWmjN5bwWD80P0CVYP6-mT5sX6BH3AR1xNrOApPF9X0plwVD4_AsWzVo435j1AmgBzPwIjhHPKtxXycaKEwSEHYFesyi-XCEJtaQZZVcjOJOs-12L2ZJeM_csk9EqKKSx0oj3jj6BciqBnLn6_hK9sEtoGenEVWEdOpkjRQBxk1m-rVZNY2IvxXMuj9C7jGXv_Sn3cU5w6arXWUsdoQtYTl5tmuF15nkD3DneT42JzNIqBkz3S2_E1VE7S1XPCje1paL7MTf58Gdg7a5FJfVlBz-bOLM-YQD0FxjGsGh1zseh7hJ7Sv8MqpH62bWnGjLnKfHqBA"
```

Save the JWT for Bob (ops team):

```bash
export BOB_JWT="eyJhbGciOiJSUzI1NiIsImtpZCI6IjU4OTE2NDUwMzIxNTk4OTQzODMiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJzb2xvLmlvIiwic3ViIjoiYm9iIiwidGVhbSI6Im9wcyIsImV4cCI6MjA3MzY3MDQ4MiwibmJmIjoxNzY2MDg2NDgyLCJpYXQiOjE3NjYwODY0ODJ9.jnRqMp3f-S0YHZEf7hPTtpWCHJRpvxqCXqZPDAzBlygdY5cC3LJ0CbGfKCVPEYrDW27mEqh4Dk6GLo1b_YvhAE5HqnUBp87D2x0bxyO_GhZv6DJKWIV14-aaYMj7J5Q3S55PLWiaT1hGjrQ4RFHWI6ol-4aCNnBUBh6VT16J1mDgnfrXN-DjIGP0u34fGNn7i8u3lK3pBKUqJK-5RfuKjAxQc-BtRpnbSHm2P3WHGPT63RR8tTwWr37_TUQY0N4cWVnlR2VHBV4jGkNbLERmSuq-FJ-ue8CY_MWq_mnKVs3tnlqD7DBRX2iA4h-HfNVJP_qlGz7DOZOqPeq0XgPyv8w"
```

### Create the JWT Validation Policy

First, create an `AgentgatewayPolicy` to validate JWT tokens on the gateway:

```bash
kubectl apply -f- <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: jwt-auth
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: mcp-gateway
  jwt:
    providers:
    - name: solo-jwt
      issuer: solo.io
      jwks:
        local:
          key: |
            {
              "keys": [
                {
                  "kty": "RSA",
                  "kid": "58916450321598943383",
                  "n": "4ixh8MjCGo3VktYy66DFnj2blFfxPHMHnh84VKFSZrXJFN0pPDuXfMx4iGmTKFWwJBBLJO0G7VKDiSJqIr0STMXZ3G3IG-nFBIjJJHR3OQkEUkqC3RlWOQy9lVhcmEp4zGqsEqNyCTMZ_4_hSVnOKhQ_8fYSTpfRbtKfhVwTjKI0vSYGPZvD4JN-LTJ2xMvDOPq-9TS6iT7SuFRoTH5dFvs4m7jEgP-Bxq5ZFpzSJFq-FsBvMnq-SyrNpAx3S3aEHhmfJBp3YsBB3sJfLQFbV7Oln-pTbfI3uQySxeCmWfMLT-YEuNwK8vsExjfWzI9LdvTODQ7BEstJVQQaaDqRiw",
                  "e": "AQAB",
                  "alg": "RS256",
                  "use": "sig"
                }
              ]
            }
EOF
```

### Test: Unauthenticated Access is Blocked

```bash
npx @modelcontextprotocol/inspector@latest \
  --cli http://localhost:8080/mcp \
  --transport http \
  --header "mcp-protocol-version: 2024-11-05" \
  --method tools/list
```

This should fail with a **401 Unauthorized** — no JWT token provided.

### Test: Authenticated Access Works

```bash
npx @modelcontextprotocol/inspector@latest \
  --cli http://localhost:8080/mcp \
  --transport http \
  --header "mcp-protocol-version: 2024-11-05" \
  --header "Authorization: Bearer $ALICE_JWT" \
  --method tools/list
```

Now you should see all tools from both servers. Both Alice and Bob can access everything — we haven't restricted tools yet.

### Restrict Tool Access by User

Now for the powerful part. Let's say:
- **Alice** (dev team) should only see the fetch tool — she uses it for research
- **Bob** (ops team) should see everything — he needs full access for operations

Create a tool access policy:

```bash
kubectl apply -f- <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: tool-access-control
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: agentgateway.dev
    kind: AgentgatewayBackend
    name: mcp-federated
  mcp:
    toolAccess:
    - cel: 'jwt.sub == "bob"'
    - cel: 'jwt.sub == "alice" && mcp.tool.name.startsWith("mcp-website-fetcher")'
EOF
```

This policy says:
- If `sub=bob` → show **all** tools (no tool name filter)
- If `sub=alice` → only show tools whose name starts with `mcp-website-fetcher`

### Verify: Alice Only Sees Time Tools

```bash
npx @modelcontextprotocol/inspector@latest \
  --cli http://localhost:8080/mcp \
  --transport http \
  --header "mcp-protocol-version: 2024-11-05" \
  --header "Authorization: Bearer $ALICE_JWT" \
  --method tools/list
```

Alice should only see:
- `mcp-website-fetcher_fetch`

The `echo`, `add`, and other everything-server tools are **hidden** from Alice.

### Verify: Bob Sees All Tools

```bash
npx @modelcontextprotocol/inspector@latest \
  --cli http://localhost:8080/mcp \
  --transport http \
  --header "mcp-protocol-version: 2024-11-05" \
  --header "Authorization: Bearer $BOB_JWT" \
  --method tools/list
```

Bob sees all tools from both servers.

### Verify: Alice Can Still Call Her Allowed Tools

```bash
npx @modelcontextprotocol/inspector@latest \
  --cli http://localhost:8080/mcp \
  --transport http \
  --header "mcp-protocol-version: 2024-11-05" \
  --header "Authorization: Bearer $ALICE_JWT" \
  --method tools/call \
  --tool-name mcp-website-fetcher_fetch \
  --tool-arg url=https://agentgateway.dev
```

Alice can call the fetch tool she's authorized to use. Trying to call `mcp-server-everything-3001_echo` would be denied.

---

## Adding More MCP Servers

The beauty of label-based federation: adding a new server is just a deployment + service with the `mcp-federation: "true"` label. No AgentGateway config changes needed — the backend's label selector picks it up automatically.

## IDE Configuration with JWT Auth

When tool access control is enabled, your IDE needs to send the JWT token. Here's how:

### Cursor

```json
{
  "mcpServers": {
    "federated-tools": {
      "url": "http://localhost:8080/mcp",
      "headers": {
        "Authorization": "Bearer <your-jwt-token>"
      }
    }
  }
}
```

### Claude Code

```bash
claude mcp add federated-tools \
  --transport sse \
  --header "Authorization: Bearer <your-jwt-token>" \
  http://localhost:8080/mcp
```

## Cleanup

```bash
kubectl delete agentgatewaypolicy jwt-auth tool-access-control -n agentgateway-system
kubectl delete httproute mcp-route -n agentgateway-system
kubectl delete agentgatewaybackend mcp-federated -n agentgateway-system
kubectl delete deploy mcp-server-everything mcp-website-fetcher -n agentgateway-system
kubectl delete svc mcp-server-everything mcp-website-fetcher -n agentgateway-system
kind delete cluster --name agentgateway-mcp
```

## Conclusion

MCP multiplexing and tool access control solve two of the biggest challenges in production agentic AI environments:

1. **Server sprawl** — developers shouldn't need to configure connections to every MCP server individually. One gateway endpoint, all tools.

2. **Ungoverned tool access** — not every agent or user should see every tool. With JWT-based RBAC and CEL expressions, you get fine-grained, identity-aware tool filtering at the gateway level.

The combination is powerful: federate all your MCP servers behind AgentGateway, then use policies to control exactly who sees what. Add new servers with a label. Restrict tools with a CEL expression. No agent code changes. No client reconfiguration.

One gateway. All your tools. The right access for the right user.
