---
title: "Your First AI Route: Connecting to OpenAI with AgentGateway (Open Source)"
date: 2026-02-11
description: "Step-by-step guide to connecting open source AgentGateway to OpenAI API with cost tracking, monitoring, security best practices, and production-ready configurations."
---

# Your First AI Route: Connecting to OpenAI with AgentGateway (Open Source)

## Introduction
This is a how-to guide to setup AgentGateway and get your first AI route working with OpenAI. We'll walk through the complete setup from scratch - creating a Kubernetes cluster, installing AgentGateway, and connecting it to OpenAI's API.

## What You'll Learn

- Create secure API key storage for OpenAI
- Configure an AgentgatewayBackend for OpenAI
- Set up HTTPRoutes for different OpenAI endpoints
- Test chat completions, embeddings, and model listings
- Monitor real AI requests through Grafana
- Understand token usage and cost tracking
- Compare mock vs real provider behavior

## Prerequisites

- Docker installed and running
- kubectl CLI tool
- Helm 3.x installed  
- Valid OpenAI API Key with credits (get from [OpenAI Platform](https://platform.openai.com))
- Basic understanding of Kubernetes and OpenAI API structure

## Setting Up Kind Cluster with AgentGateway

If you don't have AgentGateway running yet, let's set up a complete environment from scratch.

### Install Kind
```bash
# On macOS 
brew install kind

# On Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
```

### Create Kind Cluster
Create a simple kind cluster for AgentGateway:

```bash
# Create the cluster
kind create cluster --name agentgateway

# Verify cluster is ready
kubectl get nodes
```

### Install Gateway API and AgentGateway
```bash
# 1. Install Gateway API CRDs (version 1.4.0)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# 2. Install AgentGateway CRDs
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v2.2.0 agentgateway-crds \
  oci://ghcr.io/kgateway-dev/charts/agentgateway-crds

# 3. Install AgentGateway control plane
helm upgrade -i --namespace agentgateway-system \
  --version v2.2.0 agentgateway \
  oci://ghcr.io/kgateway-dev/charts/agentgateway

# 4. Verify installation
kubectl get pods -n agentgateway-system
kubectl get gatewayclass
```

## Environment Setup

### Prepare Your OpenAI API Key
First, ensure you have a valid OpenAI API key with appropriate permissions:

- **Visit OpenAI Platform**: Go to [platform.openai.com](https://platform.openai.com)
- **Create API Key**: Navigate to API Keys section and create a new key
- **Set Usage Limits**: Configure spending limits to control costs
- **Note the Key**: Copy your API key securely

### Set Environment Variables
```bash
# Set your OpenAI API key (replace with your actual key)
export OPENAI_API_KEY="sk-your-openai-api-key-here"

# Verify the key is set correctly
echo "OpenAI API Key: ${OPENAI_API_KEY:0:10}..."

# Test the key directly (optional)
curl -s "https://api.openai.com/v1/models" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  | jq '.data[0:3] | .[].id'
```

## Creating Secure API Key Storage

### Create Kubernetes Secret
Store your OpenAI API key securely in a Kubernetes secret:

```bash
# Create secret with proper authorization header format
kubectl create secret generic openai-secret \
  -n agentgateway-system \
  --from-literal="Authorization=Bearer $OPENAI_API_KEY" \
  --dry-run=client -o yaml | kubectl apply -f -

# Verify secret creation
kubectl get secret openai-secret -n agentgateway-system

# View secret structure (without revealing the key)
kubectl get secret openai-secret -n agentgateway-system -o yaml | \
  sed 's/Authorization:.*/Authorization: [REDACTED]/'
```

### Security Best Practices
```bash
# Label the secret for better organization
kubectl label secret openai-secret -n agentgateway-system \
  provider=openai \
  purpose=api-authentication

# Verify secret permissions
kubectl auth can-i get secrets -n agentgateway-system --as=system:serviceaccount:agentgateway-system:default
```

## Configuring OpenAI Backend

### Create AgentgatewayBackend
```bash
kubectl apply -f- <<'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-all-models
  namespace: agentgateway-system
  labels:
    provider: openai
    environment: production
spec:
  # Configure OpenAI as the AI provider
  ai:
    provider:
      openai:
        # Use default OpenAI endpoint (api.openai.com)
        # Optionally specify config like:
        # config:
        #   endpoint: "https://api.openai.com"
        #   model: "gpt-4o-mini" # Default model override

  # Authentication and HTTP policies
  policies:
    auth:
      secretRef:
        name: openai-secret
    http:
      requestTimeout: 120s
EOF
```

### Verify Backend Configuration
```bash
# Check backend status
kubectl get agentgatewaybackend openai-all-models -n agentgateway-system -o yaml

# Ensure backend is accepted
kubectl get agentgatewaybackend openai-all-models -n agentgateway-system -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}'
```

## Creating OpenAI Routes

### Create Gateway
First, create the Gateway resource that the HTTPRoutes will reference:

```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway
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

### Basic Chat Completions Route
Now create the HTTPRoute that references the Gateway:

```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-chat
  namespace: agentgateway-system
  labels:
    provider: openai
    endpoint: chat-completions
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway-system

  rules:
    - matches:
        # Match requests to /openai/chat/completions
        - path:
            type: PathPrefix
            value: /openai/chat/completions

      backendRefs:
        - name: openai-all-models
          group: agentgateway.dev
          kind: AgentgatewayBackend

      # Configure timeouts for potentially long AI requests
      timeouts:
        request: "120s"

      # Transform the path to OpenAI's expected format
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
EOF
```

### Models and Embeddings Routes
```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-models
  namespace: agentgateway-system
  labels:
    provider: openai
    endpoint: models
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway-system

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai/models

      backendRefs:
        - name: openai-all-models
          group: agentgateway.dev
          kind: AgentgatewayBackend

      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/models

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-embeddings
  namespace: agentgateway-system
  labels:
    provider: openai
    endpoint: embeddings
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway-system

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai/embeddings

      backendRefs:
        - name: openai-all-models
          group: agentgateway.dev
          kind: AgentgatewayBackend

      timeouts:
        request: "60s"

      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/embeddings
EOF
```

### Verify Routes
```bash
# Check all OpenAI routes
kubectl get httproute -n agentgateway-system -l provider=openai

# Verify route status
kubectl get httproute openai-chat -n agentgateway-system -o yaml | grep -A 10 status:
```

## Testing Your OpenAI Integration

### Setup Access to Gateway (Kind Cluster)
```bash
# Port-forward AgentGateway service in background
kubectl port-forward -n agentgateway-system svc/agentgateway-proxy 8080:8080 &

# Store the process ID to kill it later
PORTFORWARD_PID=$!
echo "Port-forward running as PID: $PORTFORWARD_PID"

# Set gateway endpoint
export GATEWAY_IP="localhost"
export GATEWAY_PORT="8080"

echo "AgentGateway available at: $GATEWAY_IP:$GATEWAY_PORT"
```

### Test Chat Completions
```bash
# Test basic chat completion
curl -i "$GATEWAY_IP:$GATEWAY_PORT/openai/chat/completions" \
  -H "content-type: application/json" \
  -H "x-request-id: test-$(date +%s)" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful assistant that explains AI Gateway technology."
      },
      {
        "role": "user",
        "content": "What are the key benefits of using an AI Gateway like AgentGateway for managing LLM requests?"
      }
    ],
    "max_tokens": 200,
    "temperature": 0.7
  }'
```

**Expected response:**
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
        "content": "An AI Gateway like AgentGateway provides several key benefits for managing LLM requests:\n\n1. **Unified Access**: Single endpoint for multiple AI providers (OpenAI, Anthropic, AWS Bedrock)\n2. **Observability**: Comprehensive metrics, logging, and tracing of AI requests\n3. **Cost Management**: Token usage tracking and cost attribution across teams\n4. **Security**: Centralized authentication, API key management, and access control\n5. **Rate Limiting**: Protect against excessive usage and manage quotas\n6. **Reliability**: Failover between providers, retry policies, and circuit breakers\n\nThis centralized approach simplifies AI operations while providing enterprise-grade governance."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 45,
    "completion_tokens": 156,
    "total_tokens": 201
  },
  "system_fingerprint": "fp_8bda4d3a2c"
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
        "content": "Explain the difference between GPT-4o and GPT-4o-mini in one sentence."
      }
    ],
    "max_tokens": 50
  }' | jq '.choices[0].message.content'

# Test with different temperature settings
curl -s "$GATEWAY_IP:$GATEWAY_PORT/openai/chat/completions" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "Write a creative haiku about AI gateways."
      }
    ],
    "temperature": 1.2,
    "max_tokens": 50
  }' | jq '.choices[0].message.content'
```

### Test Embeddings
```bash
# Test text embeddings
curl -s "$GATEWAY_IP:$GATEWAY_PORT/openai/embeddings" \
  -H "content-type: application/json" \
  -d '{
    "input": "AgentGateway provides unified access to multiple AI providers with enterprise-grade security and observability.",
    "model": "text-embedding-3-small"
  }' | jq '{
    model: .model,
    usage: .usage,
    embedding_length: (.data[0].embedding | length),
    first_few_values: (.data[0].embedding[0:5])
  }'
```

### Test Models List
```bash
# List available models
curl -s "$GATEWAY_IP:$GATEWAY_PORT/openai/models" | jq '.data[] | select(.id | contains("gpt")) | .id'
```

## Monitoring Real AI Requests

### View Metrics in Grafana

1. **Open Grafana:**
   ```bash
   kubectl port-forward -n monitoring svc/grafana-prometheus 3000:3000 &
   echo "Grafana: http://localhost:3000 (admin / $GRAFANA_ADMIN_PASSWORD)"
   ```

2. **Navigate to AgentGateway Dashboard:**
   - Go to Dashboards > AgentGateway Overview
   - Send several test requests
   - Observe real-time metrics:
     - Request rates by model
     - Token usage (input/output)
     - Response latencies
     - Cost estimates

3. **Key Metrics to Watch:**
   - **Request Volume**: Requests per second by model
   - **Token Consumption**: Input vs output tokens
   - **Cost Tracking**: Real-time cost accumulation
   - **Latency Percentiles**: P50, P95, P99 response times
   - **Error Rates**: Failed requests and reasons

### Analyze Distributed Traces

1. Navigate to Explore in Grafana
2. Select Tempo as data source
3. Search for traces:
   ```
   service.name="agentgateway" && gen_ai.system="openai"
   ```
4. Examine trace details:
   - Total request duration
   - Time spent in AgentGateway vs OpenAI
   - Token usage attributes
   - Error information (if any)

### Real-Time Log Analysis
```bash
# View structured logs with LLM context
kubectl logs deploy/agentgateway -n agentgateway-system | \
  jq 'select(.gen_ai) | {
    timestamp: .timestamp,
    model: .gen_ai.request.model,
    prompt_tokens: .gen_ai.usage.prompt_tokens,
    completion_tokens: .gen_ai.usage.completion_tokens,
    duration: .duration
  }'

# Monitor costs in real-time
kubectl logs deploy/agentgateway -n agentgateway-system -f | \
  grep -E "(usage|cost)" --line-buffered
```

## Performance Testing

### Create Load Test for Real OpenAI
```bash
cat <<'EOF' > load-test-openai.sh
#!/bin/bash

GATEWAY_IP="${GATEWAY_IP:-localhost}"
GATEWAY_PORT="${GATEWAY_PORT:-8080}"
CONCURRENT_REQUESTS=${1:-5}
TOTAL_REQUESTS=${2:-25}
MODEL=${3:-"gpt-4o-mini"}

echo "Load testing OpenAI through AgentGateway..."
echo "Target: $GATEWAY_IP:$GATEWAY_PORT"
echo "Model: $MODEL"
echo "Concurrent: $CONCURRENT_REQUESTS"
echo "Total: $TOTAL_REQUESTS"
echo "⚠️ This will incur OpenAI API costs!"
echo ""

read -p "Continue? (y/N): " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
  echo "Cancelled."
  exit 1
fi

# Array of test prompts
PROMPTS=(
  "Explain what an AI Gateway is in 2 sentences."
  "What are the benefits of using Kubernetes for AI workloads?"
  "How does distributed tracing help with AI observability?"
  "What is the importance of rate limiting in AI services?"
  "Describe the role of authentication in AI gateways."
)

send_request() {
  local id=$1
  local prompt="${PROMPTS[$((id % ${#PROMPTS[@]}))]}"

  local start_time=$(date +%s.%N)

  local response=$(curl -s -w "%{http_code}" "$GATEWAY_IP:$GATEWAY_PORT/openai/chat/completions" \
    -H "content-type: application/json" \
    -H "x-request-id: load-test-$id" \
    -d "{
      \"model\": \"$MODEL\",
      \"messages\": [
        {\"role\": \"user\", \"content\": \"$prompt\"}
      ],
      \"max_tokens\": 50
    }")

  local end_time=$(date +%s.%N)
  local duration=$(echo "$end_time - $start_time" | bc -l)
  local http_code="${response: -3}"

  if [ "$http_code" = "200" ]; then
    local tokens=$(echo "${response%???}" | jq -r '.usage.total_tokens // 0')
    printf "Request %3d: ✓ %s (%.2fs, %d tokens)\n" $id $http_code $duration $tokens
  else
    printf "Request %3d: ✗ %s (%.2fs)\n" $id $http_code $duration
  fi
}

echo "Starting load test..."
start_time=$(date +%s)

for ((i=1; i<=TOTAL_REQUESTS; i++)); do
  send_request $i &

  # Control concurrency
  if ((i % CONCURRENT_REQUESTS == 0)); then
    wait
  fi
done

wait
end_time=$(date +%s)
total_duration=$((end_time - start_time))

echo ""
echo "Load test complete!"
echo "Total time: ${total_duration}s"
echo "Average rate: $(echo "scale=2; $TOTAL_REQUESTS / $total_duration" | bc) req/s"
echo ""
echo "Check Grafana for detailed metrics and cost tracking."
EOF

chmod +x load-test-openai.sh

# Run a small test (optional - will incur API costs)
# ./load-test-openai.sh 3 10 "gpt-4o-mini"
```

## Cost Tracking and Management

### Understanding Token Costs
```bash
# Create cost calculation script
cat <<'EOF' > calculate-costs.sh
#!/bin/bash

# OpenAI pricing (as of 2024 - verify current pricing)
GPT4O_MINI_INPUT_COST=0.000150 # per 1K tokens
GPT4O_MINI_OUTPUT_COST=0.000600 # per 1K tokens
GPT4O_INPUT_COST=0.0025 # per 1K tokens
GPT4O_OUTPUT_COST=0.0100 # per 1K tokens
EMBEDDING_COST=0.000020 # per 1K tokens

echo "Analyzing recent token usage..."

# Extract token usage from logs
kubectl logs deploy/agentgateway -n agentgateway-system --tail=100 | \
  jq -r 'select(.gen_ai.usage) | [
    .timestamp,
    .gen_ai.request.model,
    .gen_ai.usage.prompt_tokens,
    .gen_ai.usage.completion_tokens,
    .gen_ai.usage.total_tokens
  ] | @csv' | \
  awk -F',' '
BEGIN {
  print "Timestamp,Model,Input Tokens,Output Tokens,Total Tokens,Estimated Cost"
  total_cost = 0
}
{
  model = $2
  input = $3
  output = $4

  # Remove quotes
  gsub(/"/, "", model)

  cost = 0
  if (model ~ /gpt-4o-mini/) {
    cost = (input * 0.000150 / 1000) + (output * 0.000600 / 1000)
  } else if (model ~ /gpt-4o/) {
    cost = (input * 0.0025 / 1000) + (output * 0.0100 / 1000)
  } else if (model ~ /embedding/) {
    cost = ((input + output) * 0.000020 / 1000)
  }

  total_cost += cost
  printf "%s,%s,%d,%d,%d,$%.6f\n", $1, model, input, output, input+output, cost
}
END {
  printf "\nTotal estimated cost: $%.6f\n", total_cost
}'
EOF

chmod +x calculate-costs.sh
./calculate-costs.sh
```

### Set Up Cost Alerts
```bash
# Create a simple cost monitoring script:
cat <<'EOF' > cost-monitor.sh
#!/bin/bash

COST_THRESHOLD=${1:-1.00} # Alert if costs exceed $1.00
CHECK_INTERVAL=${2:-300} # Check every 5 minutes

echo "Monitoring OpenAI costs through AgentGateway..."
echo "Threshold: \$$COST_THRESHOLD"
echo "Check interval: ${CHECK_INTERVAL}s"

while true; do
  # Calculate recent costs (last hour)
  RECENT_COST=$(kubectl logs deploy/agentgateway -n agentgateway-system --since=1h | \
    jq -r 'select(.gen_ai.usage) | .gen_ai.usage' | \
    awk '{
      if ($0 ~ /gpt-4o-mini/) cost += 0.0008
      else if ($0 ~ /gpt-4o/) cost += 0.015
      else cost += 0.0002
    } END {printf "%.4f", cost}')

  echo "$(date): Recent cost: \$$RECENT_COST"

  if (( $(echo "$RECENT_COST > $COST_THRESHOLD" | bc -l) )); then
    echo "🚨 ALERT: Cost threshold exceeded! \$$RECENT_COST > \$$COST_THRESHOLD"
    # Add notification logic here (email, Slack, etc.)
  fi

  sleep $CHECK_INTERVAL
done
EOF

chmod +x cost-monitor.sh
# Run in background: ./cost-monitor.sh 0.50 &
```

## Comparing Mock vs Real Provider

### Run Comparison Test
```bash
cat <<'EOF' > compare-mock-real.sh
#!/bin/bash

GATEWAY_IP="${GATEWAY_IP:-localhost}"
TEST_PROMPT="What are three key features of an AI Gateway?"

echo "Comparing Mock vs Real OpenAI responses..."
echo "Test prompt: $TEST_PROMPT"
echo ""

# Function to test an endpoint
test_endpoint() {
  local endpoint=$1
  local label=$2

  echo "=== $label ==="

  local start_time=$(date +%s.%N)

  local response=$(curl -s "$GATEWAY_IP:8080/$endpoint/chat/completions" \
    -H "content-type: application/json" \
    -H "authorization: bearer mock-token" \
    -d "{
      \"model\": \"gpt-4o-mini\",
      \"messages\": [
        {\"role\": \"user\", \"content\": \"$TEST_PROMPT\"}
      ],
      \"max_tokens\": 100
    }")

  local end_time=$(date +%s.%N)
  local duration=$(echo "scale=3; $end_time - $start_time" | bc -l)

  echo "Response time: ${duration}s"
  echo "Content: $(echo "$response" | jq -r '.choices[0].message.content')"
  echo "Tokens: $(echo "$response" | jq '.usage')"
  echo ""
}

# Test both endpoints
test_endpoint "mock-openai" "Mock OpenAI"
test_endpoint "openai" "Real OpenAI"

echo "Key Differences:"
echo "- Mock: Deterministic, no cost, instant response"
echo "- Real: AI-generated, costs money, variable latency"
echo "- Both: Provide observability data for monitoring"
EOF

chmod +x compare-mock-real.sh
# ./compare-mock-real.sh
```

## Troubleshooting Real Provider Issues

### Common Issues and Solutions

**1. Authentication Errors (401)**
```bash
# Verify secret exists and is properly formatted
kubectl get secret openai-secret -n agentgateway-system -o yaml

# Test API key directly
curl -s "https://api.openai.com/v1/models" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  | jq '.data[0].id'

# Check AgentGateway logs for auth errors
kubectl logs deploy/agentgateway -n agentgateway-system | grep -i auth
```

**2. Rate Limit Errors (429)**
```bash
# Check for rate limit errors in logs
kubectl logs deploy/agentgateway -n agentgateway-system | grep -i "rate\|429"

# Implement backoff in requests
curl -i "$GATEWAY_IP:8080/openai/chat/completions" \
  -H "content-type: application/json" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "test"}]}'
```

**3. Model Not Found (404)**
```bash
# List available models
curl -s "$GATEWAY_IP:8080/openai/models" | jq '.data[] | .id'

# Try with a known working model
curl -s "$GATEWAY_IP:8080/openai/chat/completions" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "test"}]
  }'
```

**4. Timeout Issues**
```bash
# Check backend timeout configuration
kubectl get agentgatewaybackend openai-all-models -n agentgateway-system -o yaml | grep -A 5 timeout

# Increase timeout if needed
kubectl patch agentgatewaybackend openai-all-models -n agentgateway-system --type='merge' -p='{
  "spec": {
    "policies": {
      "timeout": {
        "request": "180s"
      }
    }
  }
}'
```

### Debug Connection Issues
```bash
# Test connectivity from AgentGateway pod
kubectl exec -n agentgateway-system deploy/agentgateway -- \
  curl -v https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"

# Check DNS resolution
kubectl exec -n agentgateway-system deploy/agentgateway -- \
  nslookup api.openai.com

# Verify backend status
kubectl describe agentgatewaybackend openai-all-models -n agentgateway-system
```

## Environment Management
Create a helper script for managing your OpenAI integration:

```bash
cat <<'EOF' > openai-management.sh
#!/bin/bash

case $1 in
  "test")
    echo "Testing OpenAI integration..."
    curl -s "${GATEWAY_IP:-localhost}:8080/openai/chat/completions" \
      -H "content-type: application/json" \
      -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "Hello, test message"}]}' | \
      jq '.choices[0].message.content'
    ;;
  "models")
    echo "Available OpenAI models:"
    curl -s "${GATEWAY_IP:-localhost}:8080/openai/models" | jq -r '.data[] | select(.id | test("gpt|embedding")) | .id'
    ;;
  "status")
    echo "=== OpenAI Backend Status ==="
    kubectl get agentgatewaybackend openai-all-models -n agentgateway-system
    echo ""
    echo "=== OpenAI Routes ==="
    kubectl get httproute -n agentgateway-system -l provider=openai
    ;;
  "logs")
    kubectl logs deploy/agentgateway -n agentgateway-system | grep -i openai | tail -20
    ;;
  "costs")
    echo "Calculating recent costs..."
    ./calculate-costs.sh
    ;;
  "cleanup")
    echo "Removing OpenAI configuration..."
    kubectl delete httproute -n agentgateway-system -l provider=openai
    kubectl delete agentgatewaybackend openai-all-models -n agentgateway-system
    kubectl delete secret openai-secret -n agentgateway-system
    ;;
  *)
    echo "Usage: $0 {test|models|status|logs|costs|cleanup}"
    echo ""
    echo "Commands:"
    echo "  test - Send test request to OpenAI"
    echo "  models - List available models"
    echo "  status - Show backend and route status"
    echo "  logs - Show recent OpenAI-related logs"
    echo "  costs - Calculate token costs"
    echo "  cleanup - Remove all OpenAI configuration"
    ;;
esac
EOF

chmod +x openai-management.sh
```

## Stop Port-Forward

When done testing, stop the port-forward process:
```bash
# Kill the port-forward process
kill $PORTFORWARD_PID
```

## Installation with Open Source Helm Chart

To install or upgrade AgentGateway using the open source Helm chart:

```bash
# Add the open source Helm repository
helm repo add agentgateway oci://ghcr.io/agentgateway-dev/helm-charts/agentgateway

# Install AgentGateway in the agentgateway-system namespace
helm upgrade --install agentgateway agentgateway/agentgateway \
  --namespace agentgateway-system \
  --create-namespace \
  --set gateway.className=agentgateway

# Verify installation
kubectl get pods -n agentgateway-system
kubectl get gateway -n agentgateway-system
```

## Next Steps

With your real OpenAI integration working, you're ready to:

- **Add more providers** - Configure Anthropic, AWS Bedrock, or Azure OpenAI
- **Implement security** - Add authentication, rate limiting, and guardrails  
- **Explore advanced routing** - Path-based, header-based, and weighted routing
- **Set up production monitoring** - Alerts, dashboards, and cost controls

In our next blog post, we'll explore advanced routing patterns that allow you to route requests to different models or providers based on various criteria.

## Key Takeaways

- Real AI integration requires proper API key management and security
- Observability provides immediate insights into costs, performance, and usage
- AgentGateway adds minimal latency while providing significant value
- Production considerations include timeouts, retries, and cost monitoring
- Structured logging and metrics enable comprehensive AI operations visibility

Your AgentGateway now connects to production AI services while maintaining enterprise-grade security, observability, and cost control!