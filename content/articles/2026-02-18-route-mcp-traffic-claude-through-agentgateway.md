---
title: "Route MCP and LLM Traffic from Claude Desktop and Claude Code Through AgentGateway"
publishDate: 2026-02-18
author: "Sebastian Maniak"
description: "How to proxy MCP server traffic from Claude Desktop and LLM API calls from Claude Code through Solo AgentGateway for security, observability, and rate limiting — using Google Gemini as the backend provider."
---

## Introduction

Claude Desktop and Claude Code are powerful AI tools, but out of the box they talk directly to backend services with no visibility, no security controls, and no usage governance. Every MCP tool call from Claude Desktop hits your servers unmonitored. Every LLM API call from Claude Code goes straight to the provider with no rate limiting or audit trail.

What if you could put a gateway in front of all that traffic?

This guide shows you how to route both **MCP server traffic from Claude Desktop** and **LLM API calls from Claude Code** through [Solo AgentGateway](https://agentgateway.dev) — giving you JWT authentication, observability traces, rate limiting, and centralized API key management. We'll use **Google Gemini** as the LLM provider.

## What You Get

By routing traffic through AgentGateway, you gain:

- **Security**: JWT authentication on MCP endpoints, RBAC for tool access, prompt injection guards
- **Observability**: OpenTelemetry traces for every MCP tool call and LLM request
- **Rate Limiting**: Token-based and request-based limits per user or team
- **Centralized Secrets**: API keys live in Kubernetes secrets, not on developer laptops
- **Audit Trail**: Full logging of every interaction for compliance

## Architecture

```
┌──────────────────┐          ┌──────────────────────────┐
│  Claude Desktop  │          │                          │
│  (MCP traffic)   │─────────▶│                          │──▶ MCP Servers
│                  │          │                          │    (math, github, etc.)
└──────────────────┘          │    Solo AgentGateway     │
                              │    (Gateway API)         │
┌──────────────────┐          │                          │
│  Claude Code     │          │  • JWT Auth              │
│  (LLM traffic)   │─────────▶│  • Rate Limiting         │──▶ Google Gemini
│                  │          │  • OTel Tracing          │
└──────────────────┘          │  • Prompt Guards         │
                              └──────────────────────────┘
```

Claude Desktop connects to MCP servers *through* the gateway. Claude Code sends its LLM API calls *through* the gateway to Gemini. Both streams get the same security and observability treatment.

## Prerequisites

- Kubernetes cluster with AgentGateway deployed ([quickstart](https://docs.solo.io/agentgateway/latest/quickstart/))
- `kubectl` and `helm` installed
- Google Gemini API key
- Claude Desktop installed (for MCP routing)
- Claude Code CLI installed (for LLM routing): `npm install -g @anthropic-ai/claude-code`

## Part 1: Deploy an MCP Server

We'll deploy a simple math MCP server that Claude Desktop can call through the gateway.

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: mcp-math-script
  namespace: default
data:
  server.py: |
    import uvicorn
    from mcp.server.fastmcp import FastMCP
    from starlette.applications import Starlette
    from starlette.routing import Route
    from starlette.requests import Request
    from starlette.responses import JSONResponse, Response

    mcp = FastMCP("Math-Service")

    @mcp.tool()
    def add(a: int, b: int) -> int:
        return a + b

    @mcp.tool()
    def multiply(a: int, b: int) -> int:
        return a * b

    async def handle_mcp(request: Request):
        try:
            data = await request.json()
            method = data.get("method")
            msg_id = data.get("id")
            result = None

            if method == "initialize":
                result = {
                    "protocolVersion": "2024-11-05",
                    "capabilities": {"tools": {}},
                    "serverInfo": {"name": "Math-Service", "version": "1.0"}
                }
            elif method == "notifications/initialized":
                return Response(status_code=202)
            elif method == "tools/list":
                tools_list = await mcp.list_tools()
                result = {
                    "tools": [
                        {"name": t.name, "description": t.description, "inputSchema": t.inputSchema}
                        for t in tools_list
                    ]
                }
            elif method == "tools/call":
                params = data.get("params", {})
                name = params.get("name")
                args = params.get("arguments", {})
                tool_result = await mcp.call_tool(name, args)
                serialized = []
                for content in tool_result:
                    if hasattr(content, "type") and content.type == "text":
                        serialized.append({"type": "text", "text": content.text})
                    else:
                        serialized.append(content if isinstance(content, dict) else str(content))
                result = {"content": serialized, "isError": False}
            elif method == "ping":
                result = {}
            else:
                return JSONResponse(
                    {"jsonrpc": "2.0", "id": msg_id, "error": {"code": -32601, "message": "Method not found"}},
                    status_code=404
                )
            return JSONResponse({"jsonrpc": "2.0", "id": msg_id, "result": result})
        except Exception as e:
            import traceback
            traceback.print_exc()
            return JSONResponse(
                {"jsonrpc": "2.0", "id": None, "error": {"code": -32603, "message": str(e)}},
                status_code=500
            )

    app = Starlette(routes=[
        Route("/mcp", handle_mcp, methods=["POST"]),
        Route("/", lambda r: JSONResponse({"status": "ok"}), methods=["GET"])
    ])

    if __name__ == "__main__":
        uvicorn.run(app, host="0.0.0.0", port=8000)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-math-server
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-math-server
  template:
    metadata:
      labels:
        app: mcp-math-server
    spec:
      containers:
      - name: math
        image: python:3.11-slim
        command: ["/bin/sh", "-c"]
        args:
        - |
          pip install "mcp[cli]" uvicorn starlette &&
          python /app/server.py
        ports:
        - containerPort: 8000
        volumeMounts:
        - name: script-volume
          mountPath: /app
        readinessProbe:
          httpGet:
            path: /
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: script-volume
        configMap:
          name: mcp-math-script
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-math-server
  namespace: default
spec:
  selector:
    app: mcp-math-server
  ports:
  - port: 80
    targetPort: 8000
EOF
```

Wait for the pod to be ready:

```bash
kubectl wait --for=condition=ready pod -l app=mcp-math-server --timeout=120s
```

## Part 2: Create the Gateway and Routes

We need two things routed through AgentGateway: MCP tool traffic and LLM API traffic to Gemini.

### Create the Gemini API Key Secret

```bash
kubectl create secret generic gemini-api-key \
  -n agentgateway-system \
  --from-literal=Authorization="Bearer $GEMINI_API_KEY"
```

### Create the Gateway

A single Gateway with one listener handles both MCP and LLM traffic:

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ai-gateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - name: http
    port: 8080
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: All
```

### Create the MCP Backend and Route

Point the gateway at our math MCP server:

```yaml
# mcp-backend.yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: math-mcp
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: math-service
      static:
        host: mcp-math-server.default.svc.cluster.local
        port: 80
        path: /mcp
        protocol: StreamableHTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp-route
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: ai-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mcp
    backendRefs:
    - name: math-mcp
      group: agentgateway.dev
      kind: AgentgatewayBackend
```

### Create the Gemini LLM Backend and Route

```yaml
# gemini-backend.yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: gemini-llm
  namespace: agentgateway-system
spec:
  type: llm
  llm:
    provider:
      gemini:
        authToken:
          secretRef:
            name: gemini-api-key
            namespace: agentgateway-system
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gemini-route
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: ai-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /gemini
    backendRefs:
    - name: gemini-llm
      group: agentgateway.dev
      kind: AgentgatewayBackend
```

Apply everything:

```bash
kubectl apply -f gateway.yaml
kubectl apply -f mcp-backend.yaml
kubectl apply -f gemini-backend.yaml
```

### Get the Gateway Address

```bash
export GATEWAY_IP=$(kubectl get svc ai-gateway -n agentgateway-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Gateway: http://$GATEWAY_IP:8080"
```

For local clusters (kind/minikube), use port-forward instead:

```bash
kubectl port-forward -n agentgateway-system svc/ai-gateway 8080:8080 &
export GATEWAY_IP=localhost
```

## Part 3: Configure Claude Desktop (MCP Traffic)

Claude Desktop can route its MCP tool calls through AgentGateway using [supergateway](https://github.com/nichochar/supergateway) as a local bridge.

### Config File Location

- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

### Basic Configuration

Create or update the config file:

```json
{
  "mcpServers": {
    "math-service": {
      "command": "npx",
      "args": ["-y", "supergateway", "--streamableHttp", "http://GATEWAY_IP:8080/mcp"]
    }
  }
}
```

Replace `GATEWAY_IP` with your actual gateway IP or `localhost` if using port-forward.

### With JWT Authentication

If you've configured JWT auth on the gateway:

```json
{
  "mcpServers": {
    "math-service": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://GATEWAY_IP:8080/mcp"],
      "env": {
        "MCP_HEADERS": "Authorization: Bearer <your-jwt-token>"
      }
    }
  }
}
```

Restart Claude Desktop after saving the config. You should see the math tools available in the tools menu.

## Part 4: Configure Claude Code (LLM Traffic via Gemini)

Claude Code can route its LLM API traffic through AgentGateway to Gemini. This means your API keys stay in Kubernetes secrets — not on developer machines.

### Set the Base URL

Point Claude Code at the gateway's Gemini route:

```bash
export ANTHROPIC_BASE_URL=http://$GATEWAY_IP:8080/gemini
```

For persistence, add it to your shell profile:

```bash
echo "export ANTHROPIC_BASE_URL=http://$GATEWAY_IP:8080/gemini" >> ~/.zshrc
```

### Run Claude Code

```bash
claude
```

All LLM traffic now flows through AgentGateway to Gemini. You can verify by checking the gateway logs:

```bash
kubectl logs -n agentgateway-system -l gateway.networking.k8s.io/gateway-name=ai-gateway -f
```

## Part 5: Add Security Policies

Now that traffic flows through the gateway, you can layer on security controls.

### Rate Limiting

Prevent runaway costs with token-based rate limiting:

```yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: rate-limit
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: gemini-route
  default:
    rateLimiting:
      tokenBucket:
        maxTokens: 10000
        refillRate: 1000
        refillInterval: 60s
```

### Prompt Guards

Block prompt injection attempts before they reach Gemini:

```yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: prompt-guard
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: gemini-route
  default:
    promptGuard:
      request:
        matches:
        - action: REJECT
          regex: "ignore (previous|all) instructions"
```

Apply the policies:

```bash
kubectl apply -f rate-limit.yaml
kubectl apply -f prompt-guard.yaml
```

## Part 6: Enable Observability

Add OpenTelemetry tracing to see every request in detail. If you followed our [Langfuse observability guide](2026-02-18-llm-observability-agentgateway-langfuse), you already have this set up. Otherwise, configure tracing via Helm values:

```yaml
# values-tracing.yaml
gateway:
  envs:
    OTEL_EXPORTER_OTLP_ENDPOINT: "http://your-otel-collector:4317"
    OTEL_EXPORTER_OTLP_PROTOCOL: "grpc"
```

```bash
helm upgrade agentgateway oci://cr.agentgateway.dev/helm/agentgateway \
  --version 2.1.0 \
  --namespace agentgateway-system \
  -f values-tracing.yaml
```

With tracing enabled, every Claude Desktop tool call and every Claude Code LLM request shows up as a trace with full metadata: model, tokens, latency, route, and any security policy actions.

## Testing the Full Setup

### Test MCP (Claude Desktop)

Open Claude Desktop and ask it to use the math tools:

> "What's 42 multiplied by 17?"

Claude should invoke the `multiply` tool through AgentGateway. Check the gateway logs to confirm the request was proxied.

### Test LLM (Claude Code)

```bash
ANTHROPIC_BASE_URL=http://$GATEWAY_IP:8080/gemini claude

# In Claude Code, type any prompt — it routes through the gateway to Gemini
```

### Test with MCP Inspector

You can also verify MCP connectivity directly:

```bash
npx @modelcontextprotocol/inspector
```

Enter the URL: `http://GATEWAY_IP:8080/mcp`

You should see the math tools listed and be able to invoke them.

## Important Limitations

- **Claude Desktop's core LLM traffic** (conversations with Claude itself) cannot be routed through AgentGateway — only MCP server traffic is proxied
- **Claude Code LLM traffic** *can* be fully routed through the gateway via the `ANTHROPIC_BASE_URL` environment variable
- For local development (kind/minikube), you'll need port-forwarding since there's no external load balancer

## Troubleshooting

**MCP tools not showing in Claude Desktop?**
- Restart Claude Desktop after config changes
- Verify the gateway is reachable: `curl http://GATEWAY_IP:8080/mcp`
- Check that supergateway/mcp-remote is installed: `npx -y supergateway --help`

**Claude Code connection refused?**
- Verify the gateway service has an external IP: `kubectl get svc -n agentgateway-system`
- Check firewall rules allow traffic on port 8080
- Ensure `ANTHROPIC_BASE_URL` is set correctly: `echo $ANTHROPIC_BASE_URL`

**Timeout errors?**
- LLM requests can take time — ensure gateway timeouts are configured for AI workloads
- Check gateway pod resources are sufficient for the traffic volume

**Authentication errors?**
- Verify the Gemini API key secret exists: `kubectl get secret gemini-api-key -n agentgateway-system`
- Check the key is valid with a direct curl to Gemini's API

## Cleanup

```bash
kubectl delete -f gateway.yaml -f mcp-backend.yaml -f gemini-backend.yaml
kubectl delete deployment mcp-math-server
kubectl delete svc mcp-math-server
kubectl delete configmap mcp-math-script
kubectl delete secret gemini-api-key -n agentgateway-system
```

## Resources

- [Solo AgentGateway Docs](https://docs.solo.io/agentgateway/)
- [AgentGateway OSS](https://agentgateway.dev)
- [Claude Desktop MCP Documentation](https://docs.anthropic.com/en/docs/claude-desktop/mcp)
- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
- [Original demo configs](https://github.com/AdminTurnedDevOps/agentic-demo-repo/tree/main/agent-desktop-configs)
- [Google Gemini API](https://ai.google.dev/)
