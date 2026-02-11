---
title: "Your First AI Route: Connecting to OpenAI with AgentGateway (Open Source)"
date: 2026-02-11
description: "Step-by-step guide to connecting open source AgentGateway to OpenAI API with cost tracking, monitoring, security best practices, and production-ready configurations."
---

# Your First AI Route: Connecting to OpenAI with AgentGateway (Open Source)

## Introduction
This is a how-to guide to setup AgentGateway and get your first AI route working with OpenAI. We'll walk through the complete setup from scratch - creating a Kubernetes cluster, installing AgentGateway, and connecting it to OpenAI's API.

## What You'll Learn

- Create a Kubernetes cluster and install AgentGateway
- Set up secure OpenAI API key storage
- Configure AgentGateway to route to OpenAI
- Test chat completions, embeddings, and model listings
- Monitor real AI requests and track costs
- Troubleshoot common issues

## Prerequisites

- Docker installed and running
- kubectl CLI tool
- Helm 3.x installed  
- Valid OpenAI API Key with credits (get from [OpenAI Platform](https://platform.openai.com))
- Basic understanding of Kubernetes and OpenAI API structure

---

## Step 1: Environment Setup

### Install Kind
```bash
# On macOS 
brew install kind

# On Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
```

### Create Kind Cluster
```bash
# Create the cluster
kind create cluster --name agentgateway

# Verify cluster is ready
kubectl get nodes
```

### Install AgentGateway
```bash
# 1. Install Gateway API CRDs (version 1.4.0)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# 2. Install AgentGateway CRDs
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v2.2.0 agentgateway-crds \
  oci://ghcr.io/kgateway-dev/charts/agentgateway-crds

# 3. Install AgentGateway control plane
helm upgrade -i -n agentgateway-system agentgateway \
  oci://ghcr.io/kgateway-dev/charts/agentgateway \
  --version v2.2.0

# 4. Verify installation
kubectl get pods -n agentgateway-system
```

---

## Step 2: OpenAI API Key Setup

### Get Your OpenAI API Key
1. Visit [OpenAI Platform](https://platform.openai.com)
2. Navigate to API Keys section and create a new key
3. Set usage limits to control costs
4. Copy your API key securely

### Test Your API Key
```bash
# Set your OpenAI API key (replace with your actual key)
export OPENAI_API_KEY="sk-your-openai-api-key-here"

# Test the key directly
curl -s "https://api.openai.com/v1/models" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  | jq '.data[0:3] | .[].id'
```

### Create Kubernetes Secret
```bash
# Create secret with proper authorization header format
kubectl create secret generic openai-secret \
  -n agentgateway-system \
  --from-literal="Authorization=Bearer $OPENAI_API_KEY" \
  --dry-run=client -o yaml | kubectl apply -f -

# Verify secret creation
kubectl get secret openai-secret -n agentgateway-system
```

---

## Step 3: Configure AgentGateway

### Create Gateway Resource
```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - protocol: HTTP
    port: 8080
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

### Create OpenAI Backend
```bash
kubectl apply -f- <<'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-backend
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai:
        model: gpt-4o-mini
  policies:
    auth:
      secretRef:
        name: openai-secret
    http:
      requestTimeout: 120s
EOF
```

### Create HTTP Routes

First, create a backend for non-AI endpoints (like models list):

**Why two backends?** AI endpoints like `/chat/completions` expect JSON payloads, but `/models` is a simple GET request. We need different backend types to handle each properly.

```bash
kubectl apply -f- <<'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-models-backend
  namespace: agentgateway-system
spec:
  policies:
    auth:
      secretRef:
        name: openai-secret
    http:
      requestTimeout: 30s
  static:
    host: api.openai.com
    port: 443
EOF
```

Then create the HTTP routes:
```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-chat
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /openai/chat/completions
    backendRefs:
    - name: openai-backend
      group: agentgateway.dev
      kind: AgentgatewayBackend
    timeouts:
      request: "120s"
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /v1/chat/completions
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-models
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /openai/models
    backendRefs:
    - name: openai-models-backend
      group: agentgateway.dev
      kind: AgentgatewayBackend
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /v1/models
EOF
```

### Verify Configuration
```bash
# Check both backends
kubectl get agentgatewaybackend -n agentgateway-system

# Check routes
kubectl get httproute -n agentgateway-system

# Check Gateway
kubectl get gateway agentgateway-proxy -n agentgateway-system
```

---

## Step 4: Testing Your Setup

### Setup Port-Forward
```bash
# Port-forward AgentGateway service in background
kubectl port-forward -n agentgateway-system svc/agentgateway-proxy 8080:8080 &

# Store the process ID to kill it later
PORTFORWARD_PID=$!
echo "Port-forward running as PID: $PORTFORWARD_PID"

# Set gateway endpoint
export GATEWAY_IP="localhost"
export GATEWAY_PORT="8080"
```

### Test Chat Completions
```bash
# Test basic chat completion
curl -i "$GATEWAY_IP:$GATEWAY_PORT/openai/chat/completions" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "What are the key benefits of using an AI Gateway?"
      }
    ],
    "max_tokens": 100
  }'
```

**Expected Response:**
```json
{
  "id": "chatcmpl-abc123def456",
  "object": "chat.completion",
  "created": 1701234567,
  "model": "gpt-4o-mini-2024-07-18",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "An AI Gateway provides unified access to multiple AI providers, centralized security and authentication, comprehensive observability and cost tracking, rate limiting and quotas, and improved reliability through failover and retry mechanisms."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 15,
    "completion_tokens": 35,
    "total_tokens": 50
  }
}
```

### Test Different Models
```bash
# Test with GPT-4o
curl -s "$GATEWAY_IP:$GATEWAY_PORT/openai/chat/completions" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {
        "role": "user",
        "content": "Explain AgentGateway in one sentence."
      }
    ],
    "max_tokens": 50
  }' | jq '.choices[0].message.content'
```

### Test Models List
```bash
# List available models (check raw response first)
curl -i "$GATEWAY_IP:$GATEWAY_PORT/openai/models"

# If successful, filter for GPT models
curl -s "$GATEWAY_IP:$GATEWAY_PORT/openai/models" | jq -r '.data[]? | select(.id | contains("gpt")) | .id'
```

---

## Step 5: Monitoring and Observability

### View Real-Time Logs
```bash
# Check what logs look like first
kubectl logs deploy/agentgateway -n agentgateway-system --tail=5

# View structured logs with AI context (filter JSON lines only)
kubectl logs deploy/agentgateway -n agentgateway-system --tail=50 | \
  grep '^{' | \
  jq 'select(.gen_ai?) | {
    timestamp: .timestamp,
    model: .gen_ai.request.model,
    prompt_tokens: .gen_ai.usage.prompt_tokens,
    completion_tokens: .gen_ai.usage.completion_tokens,
    duration: .duration
  }'
```

### Monitor Costs
```bash
# Create cost calculation script
cat <<'EOF' > calculate-costs.sh
#!/bin/bash

echo "Analyzing recent token usage..."

kubectl logs deploy/agentgateway -n agentgateway-system --tail=50 | \
  grep '^{' | \
  jq -r 'select(.gen_ai.usage?) | [
    .timestamp,
    .gen_ai.request.model,
    .gen_ai.usage.prompt_tokens,
    .gen_ai.usage.completion_tokens,
    .gen_ai.usage.total_tokens
  ] | @csv' | \
  awk -F',' '
BEGIN {
  print "Model,Input Tokens,Output Tokens,Total Tokens,Estimated Cost"
  total_cost = 0
}
{
  model = $2
  input = $3
  output = $4
  gsub(/"/, "", model)
  
  cost = 0
  if (model ~ /gpt-4o-mini/) {
    cost = (input * 0.000150 / 1000) + (output * 0.000600 / 1000)
  } else if (model ~ /gpt-4o/) {
    cost = (input * 0.0025 / 1000) + (output * 0.0100 / 1000)
  }
  
  total_cost += cost
  printf "%s,%d,%d,%d,$%.6f\n", model, input, output, input+output, cost
}
END {
  printf "\nTotal estimated cost: $%.6f\n", total_cost
}'
EOF

chmod +x calculate-costs.sh
./calculate-costs.sh
```

---

## Troubleshooting

### Common Issues

**1. Service Not Found Error:**
```bash
# Check what services exist
kubectl get svc -n agentgateway-system

# If agentgateway-proxy doesn't exist, use the correct service name
kubectl get svc -n agentgateway-system | grep -i gateway
```

**2. Authentication Errors (401):**
```bash
# Verify secret exists
kubectl get secret openai-secret -n agentgateway-system -o yaml

# Test API key directly
curl -s "https://api.openai.com/v1/models" \
  -H "Authorization: Bearer $OPENAI_API_KEY" | jq '.data[0].id'
```

**3. Routes Not Working:**
```bash
# Check backend status
kubectl describe agentgatewaybackend openai-backend -n agentgateway-system

# Check route status
kubectl describe httproute openai-chat -n agentgateway-system

# Check Gateway status
kubectl describe gateway agentgateway-proxy -n agentgateway-system
```

**4. Port-Forward Issues:**
```bash
# Check if port 8080 is in use
lsof -i :8080

# Try a different port
kubectl port-forward -n agentgateway-system svc/agentgateway-proxy 8081:8080 &
export GATEWAY_PORT="8081"
```

### Debug Commands
```bash
# View all AgentGateway resources
kubectl get agentgatewaybackend,gateway,httproute -n agentgateway-system

# Check pod logs for errors
kubectl logs deploy/agentgateway -n agentgateway-system --tail=20

# Test connectivity from inside cluster
kubectl exec -n agentgateway-system deploy/agentgateway -- \
  curl -v https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

---

## Cleanup

### Stop Port-Forward
```bash
# Kill the port-forward process
kill $PORTFORWARD_PID
```

### Remove Resources (Optional)
```bash
# Remove all OpenAI configuration
kubectl delete httproute openai-chat openai-models -n agentgateway-system
kubectl delete agentgatewaybackend openai-backend openai-models-backend -n agentgateway-system
kubectl delete gateway agentgateway-proxy -n agentgateway-system
kubectl delete secret openai-secret -n agentgateway-system
```

### Remove Kind Cluster (Optional)
```bash
# Delete the entire cluster
kind delete cluster --name agentgateway
```

---

## Next Steps

With your OpenAI integration working, you can now:

- **Add more providers** - Configure Anthropic, AWS Bedrock, or Azure OpenAI
- **Implement security** - Add rate limiting, authentication, and guardrails
- **Set up monitoring** - Configure Grafana dashboards and alerting
- **Explore advanced routing** - Path-based, header-based, and weighted routing

## Key Takeaways

- AgentGateway provides a unified interface to AI providers with minimal overhead
- Proper secret management is essential for production deployments
- Built-in observability gives immediate insights into costs and performance
- The Gateway API pattern makes routing configuration declarative and portable
- Kind clusters are perfect for local development and testing

Your AgentGateway is now successfully routing requests to OpenAI with enterprise-grade security and observability! 🎯