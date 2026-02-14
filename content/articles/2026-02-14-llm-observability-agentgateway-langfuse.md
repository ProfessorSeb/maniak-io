---
title: "Open Source LLM Observability: Tracing AI Calls with AgentGateway and Langfuse"
date: 2026-02-14
description: "How to get full visibility into every LLM call your AI agents make — prompts, completions, token usage, costs — without changing a line of application code. Using AgentGateway's native OpenTelemetry tracing with Langfuse."
---

Your AI agents are calling LLMs hundreds of times a day. Do you know what they're sending? What they're spending? Whether that prompt injection guard actually fired?

This guide shows how to wire [Langfuse](https://langfuse.com) into [AgentGateway](https://agentgateway.dev) to get **full observability over every LLM and MCP tool call** — zero application code changes required.

---

## Why Gateway-Level Observability?

Most teams add tracing inside their application code — wrapping LLM SDK calls with Langfuse decorators or OpenTelemetry spans. This works, but it has gaps:

- **You only see what the app reports.** If a developer forgets to instrument a call, it's invisible.
- **Gateway-level policies are opaque.** PII redaction, prompt injection blocking, rate limiting — these happen at the gateway. Application-level tracing can't see them.
- **Multiple agents, multiple codebases.** Every team has to add their own instrumentation. Different languages, different quality.

When tracing happens at the **gateway layer**, every request is captured automatically. Every agent, every provider, every tool call — same fidelity, same format, zero code changes.

---

## Architecture

```
┌──────────────┐     ┌────────────────────────┐     ┌─────────────────┐
│ Your App /   │     │   Solo AgentGateway     │     │   LLM Provider  │
│ AI Agent     │────▶│   (Gateway API)         │────▶│   (OpenAI, etc) │
│              │     │                         │     │                 │
└──────────────┘     └───────────┬────────────┘     └─────────────────┘
                                 │
                          OTLP Traces (gRPC)
                                 │
                     ┌───────────▼────────────┐
                     │  OpenTelemetry          │
                     │  Collector (fan-out)    │
                     │                         │
                     └─────┬───────────┬──────┘
                           │           │
                    OTLP HTTP      OTLP gRPC
                           │           │
                     ┌─────▼──┐  ┌─────▼──────────┐
                     │Langfuse│  │ClickHouse /     │
                     │  UI    │  │ Solo Enterprise  │
                     │        │  │ UI (optional)    │
                     └────────┘  └─────────────────┘
```

AgentGateway natively emits OpenTelemetry traces for every LLM request. A lightweight OTel Collector receives these traces and forwards them to Langfuse via OTLP HTTP. The same collector can fan-out traces to additional backends (ClickHouse, Jaeger, Datadog) simultaneously.

---

## Quick Start: Kind Cluster + AgentGateway 2.1 OSS

Don't have a cluster? Here's the fastest path from zero to traced LLM calls.

### Create the Cluster

```bash
kind create cluster --name agentgateway
```

### Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

### Install AgentGateway 2.1 OSS

```bash
helm upgrade -i agentgateway-crds oci://cr.agentgateway.dev/helm/agentgateway-crds \
  --version 2.1.0 \
  --namespace agentgateway-system \
  --create-namespace

# Install control plane
helm upgrade -i agentgateway oci://cr.agentgateway.dev/helm/agentgateway \
  --version 2.1.0 \
  --namespace agentgateway-system
```

Verify it's running:

```bash
kubectl get pods -n agentgateway-system
kubectl get gatewayclass agentgateway
```

---

## Setting Up Langfuse Tracing

### Step 1: Get Langfuse API Keys

Sign up at [cloud.langfuse.com](https://cloud.langfuse.com) (free tier) or use a self-hosted instance. Go to **Settings → API Keys** and create a key pair.

Base64 encode your credentials:

```bash
echo -n "pk-lf-YOUR_PUBLIC_KEY:sk-lf-YOUR_SECRET_KEY" | base64
```

### Step 2: Deploy the OTel Collector

The collector bridges AgentGateway's OTLP gRPC output to Langfuse's OTLP HTTP input:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: langfuse-otel-collector-config
  namespace: agentgateway-system
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    exporters:
      otlphttp/langfuse:
        endpoint: http://cloud.langfuse.com/api/public/otel
        headers:
          Authorization: "Basic <YOUR_BASE64_CREDENTIALS>"
        retry_on_failure:
          enabled: true
          initial_interval: 5s
          max_interval: 30s
          max_elapsed_time: 300s
    processors:
      batch:
        send_batch_size: 1000
        timeout: 5s
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlphttp/langfuse]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: langfuse-otel-collector
  namespace: agentgateway-system
  labels:
    app: langfuse-otel-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: langfuse-otel-collector
  template:
    metadata:
      labels:
        app: langfuse-otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:0.132.1
          args: ["--config=/conf/config.yaml"]
          ports:
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
          volumeMounts:
            - name: config
              mountPath: /conf
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
      volumes:
        - name: config
          configMap:
            name: langfuse-otel-collector-config
---
apiVersion: v1
kind: Service
metadata:
  name: langfuse-otel-collector
  namespace: agentgateway-system
spec:
  selector:
    app: langfuse-otel-collector
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
```

```bash
kubectl apply -f langfuse-collector.yaml
```

### Step 3: Configure AgentGateway Tracing

For AgentGateway OSS, enable tracing via Helm values:

```yaml
# values-tracing.yaml
gateway:
  envs:
    OTEL_EXPORTER_OTLP_ENDPOINT: "http://langfuse-otel-collector.agentgateway-system.svc.cluster.local:4317"
    OTEL_EXPORTER_OTLP_PROTOCOL: "grpc"
```

```bash
helm upgrade agentgateway oci://cr.agentgateway.dev/helm/agentgateway \
  --version 2.1.0 \
  --namespace agentgateway-system \
  -f values-tracing.yaml
```

For **AgentGateway Enterprise**, use the `EnterpriseAgentgatewayParameters` resource instead:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayParameters
metadata:
  name: tracing
  namespace: agentgateway-system
spec:
  rawConfig:
    config:
      tracing:
        otlpEndpoint: grpc://langfuse-otel-collector.agentgateway-system.svc.cluster.local:4317
        otlpProtocol: grpc
        randomSampling: true
        fields:
          add:
            gen_ai.operation.name: '"chat"'
            gen_ai.system: "llm.provider"
            gen_ai.request.model: "llm.requestModel"
            gen_ai.response.model: "llm.responseModel"
            gen_ai.usage.prompt_tokens: "llm.inputTokens"
            gen_ai.usage.completion_tokens: "llm.outputTokens"
            gen_ai.usage.total_tokens: "llm.totalTokens"
            gen_ai.request.temperature: "llm.params.temperature"
            gen_ai.prompt: "llm.prompt"
            gen_ai.completion: "llm.completion"
```

### Step 4: Create a Gateway and Route

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ai-gateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
    - name: llm
      port: 8080
      protocol: HTTPS
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: ai-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai
      backendRefs:
        - group: agentgateway.dev
          kind: AgentgatewayBackend
          name: openai
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  type: llm
  llm:
    provider:
      openai:
        authToken:
          secretRef:
            name: openai-api-key
            namespace: agentgateway-system
```

```bash
# Create the API key secret
kubectl create secret generic openai-api-key \
  -n agentgateway-system \
  --from-literal=Authorization="Bearer $OPENAI_API_KEY"

# Apply the gateway and route
kubectl apply -f gateway.yaml
```

### Step 5: Test and View Traces

```bash
# Port-forward the gateway
kubectl port-forward -n agentgateway-system svc/ai-gateway 8080:8080 &

# Send a test request
curl -X POST http://localhost:8080/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4.1-mini",
    "messages": [{"role": "user", "content": "Hello from AgentGateway!"}]
  }'
```

Open Langfuse → **Traces**. You should see a trace with model, tokens, prompt, completion, and gateway metadata.

---

## What Gets Captured

### Trace Attributes (GenAI Semantic Conventions)

| Attribute | Description | Example |
|-----------|-------------|---------|
| `gen_ai.system` | LLM provider | `openai` |
| `gen_ai.request.model` | Requested model | `gpt-4.1-mini` |
| `gen_ai.response.model` | Actual model used | `gpt-4o-mini-2024-07-18` |
| `gen_ai.usage.prompt_tokens` | Input tokens | `13` |
| `gen_ai.usage.completion_tokens` | Output tokens | `30` |
| `gen_ai.usage.total_tokens` | Combined | `43` |
| `gen_ai.prompt` | Full prompt content | `[{"role":"user","content":"Hello"}]` |
| `gen_ai.completion` | Full completion | `Hello! How can I help you?` |
| `gen_ai.streaming` | Streaming used | `false` |

### Gateway Metadata

| Attribute | Description | Example |
|-----------|-------------|---------|
| `gateway` | Gateway resource | `agentgateway-system/ai-gateway` |
| `route` | HTTPRoute name | `agentgateway-system/openai` |
| `endpoint` | Backend endpoint | `api.openai.com:443` |
| `listener` | Gateway listener | `llm` |

---

## Multi-Provider Support

AgentGateway traces all providers through the same pipeline — OpenAI, Anthropic, xAI/Grok, Azure OpenAI, Google Gemini, Ollama, and any OpenAI-compatible API. Add more routes, same observability:

```
/openai/*    → OpenAI GPT    → traced to Langfuse
/anthropic/* → Anthropic     → traced to Langfuse
/xai/*       → xAI Grok     → traced to Langfuse
```

Every provider, same trace format, same dashboard.

---

## Fan-Out: Langfuse + Additional Backends

Send traces to multiple backends simultaneously:

```yaml
exporters:
  otlphttp/langfuse:
    endpoint: http://cloud.langfuse.com/api/public/otel
    headers:
      Authorization: "Basic <CREDENTIALS>"
  otlp/jaeger:
    endpoint: jaeger-collector:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/langfuse, otlp/jaeger]
```

Common fan-out targets: Langfuse (LLM analytics) + ClickHouse (gateway metrics) + Jaeger (distributed tracing) + Datadog (enterprise monitoring).

---

## MCP Tool Tracing

AgentGateway doesn't just trace LLM calls — it also traces MCP (Model Context Protocol) tool interactions:

- **Tool discovery** (`tools/list`) — which tools are available, how long discovery takes
- **Tool execution** (`tools/call`) — parameters, results, latency
- **Backend MCP server performance** — per-server latency and error rates

When an agent calls Slack, GitHub, or any MCP tool server through AgentGateway, the full tool call chain appears in Langfuse alongside the LLM calls that triggered it.

---

## Security Policy Visibility

When AgentGateway's security policies fire, the trace metadata includes what happened:

- **PII Protection** — how many entities were redacted, what types (email, SSN, phone)
- **Prompt Injection** — whether an injection was detected and blocked
- **Credential Leak** — whether secrets were caught in the LLM response
- **Rate Limiting** — remaining quota for the user

This means you can see not just *what* your agents are doing, but *what guardrails are protecting them*.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No traces in Langfuse | Check collector pod is running: `kubectl get pods -n agentgateway-system -l app=langfuse-otel-collector` |
| 401 errors in collector logs | Wrong Langfuse API credentials — re-check the base64 encoding |
| Traces show but no prompt/completion | Add the `fields.add` section in Enterprise, or check OTEL env vars in OSS |
| Missing gateway metadata | Restart proxies after config change: `kubectl rollout restart deployment -n agentgateway-system` |

---

## Full Source Code

All manifests, configs, and examples are in the [agentgateway-langfuse](https://github.com/ProfessorSeb/agentgateway-langfuse) repository:

| Path | Description |
|------|-------------|
| [docs/quickstart-kind.md](https://github.com/ProfessorSeb/agentgateway-langfuse/blob/main/docs/quickstart-kind.md) | Full kind cluster quickstart guide |
| [examples/basic/](https://github.com/ProfessorSeb/agentgateway-langfuse/tree/main/examples/basic) | Basic Langfuse collector + tracing config |
| [examples/fan-out/](https://github.com/ProfessorSeb/agentgateway-langfuse/tree/main/examples/fan-out) | Fan-out to Langfuse + additional backends |
| [examples/argocd/](https://github.com/ProfessorSeb/agentgateway-langfuse/tree/main/examples/argocd) | Production ArgoCD/GitOps deployment |
| [scripts/verify.sh](https://github.com/ProfessorSeb/agentgateway-langfuse/blob/main/scripts/verify.sh) | End-to-end verification script |

---

## Resources

- [AgentGateway OSS](https://agentgateway.dev) — CNCF open-source AI gateway
- [AgentGateway Helm Install](https://agentgateway.dev/docs/kubernetes/latest/install/helm/)
- [Langfuse](https://langfuse.com) — Open-source LLM observability
- [Langfuse OpenTelemetry Docs](https://langfuse.com/docs/integrations/opentelemetry)
- [OpenTelemetry GenAI Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
