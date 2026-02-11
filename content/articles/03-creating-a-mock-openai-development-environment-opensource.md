---
title: "Creating a Mock OpenAI Development Environment with Open Source AgentGateway"
date: 2026-02-11
description: "Learn how to set up a realistic mock OpenAI server with open source AgentGateway for cost-effective development, testing, and load testing without spending money on real API calls."
---

# Creating a Mock OpenAI Development Environment with Open Source AgentGateway

## Introduction
Before connecting to real AI providers and spending money on API calls, it's valuable to have a local mock server that simulates OpenAI's API. This allows you to test your AgentGateway configuration, observability setup, routing patterns, and security features without incurring costs or rate limits.

In this guide, we'll deploy a mock OpenAI server that responds with realistic-looking AI responses, complete with token usage metrics and streaming support. This is perfect for development, testing, and demonstration environments using the open source AgentGateway.

## What You'll Learn

- Deploy a mock OpenAI API server in your kind cluster
- Configure open source AgentGateway to route to the mock server
- Test various OpenAI API endpoints (chat completions, embeddings)
- Understand request/response patterns before using real providers
- Use mock environments for load testing and development

## Prerequisites

- Kind cluster or any Kubernetes cluster
- kubectl and basic understanding of Kubernetes concepts
- Helm 3.x installed
- Basic understanding of Gateway API concepts

## Understanding Mock OpenAI Benefits

### Why Use a Mock Server?

- **Cost Control**: No API charges during development and testing
- **Deterministic Responses**: Predictable outputs for testing
- **Rate Limit Testing**: Simulate rate limiting without hitting real limits
- **Offline Development**: Work without internet connectivity to AI providers
- **Load Testing**: Test high traffic patterns safely
- **Feature Development**: Test new features before production

## Environment Setup

First, let's install the open source AgentGateway following the official installation steps:

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

## Deploying Mock OpenAI Server

### Create Mock OpenAI Deployment
We'll create a simple mock server that implements key OpenAI API endpoints:

```bash
kubectl apply -f- <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mock-openai
  namespace: agentgateway-system
  labels:
    app: mock-openai
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mock-openai
  template:
    metadata:
      labels:
        app: mock-openai
    spec:
      containers:
      - name: mock-openai
        image: wiremock/wiremock:3.3.1
        ports:
        - containerPort: 8080
          name: http
        args:
        - --port=8080
        - --verbose
        volumeMounts:
        - name: mock-config
          mountPath: /home/wiremock/mappings
        - name: mock-responses
          mountPath: /home/wiremock/__files
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 300m
            memory: 256Mi
      volumes:
      - name: mock-config
        configMap:
          name: mock-openai-mappings
      - name: mock-responses
        configMap:
          name: mock-openai-responses
---
apiVersion: v1
kind: Service
metadata:
  name: mock-openai
  namespace: agentgateway-system
  labels:
    app: mock-openai
spec:
  selector:
    app: mock-openai
  ports:
  - name: http
    port: 443  # Simulate HTTPS endpoint
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
EOF
```

### Create Mock API Mappings
Define the mock API endpoints that simulate OpenAI's API:

```bash
kubectl create configmap mock-openai-mappings -n agentgateway-system --from-literal=chat-completions.json='
{
  "request": {
    "method": "POST",
    "urlPath": "/v1/chat/completions",
    "headers": {
      "Content-Type": {
        "contains": "application/json"
      }
    },
    "bodyPatterns": [
      {
        "matchesJsonPath": "$.model"
      }
    ]
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json",
      "X-Request-ID": "{{randomValue type=\"UUID\"}}",
      "Access-Control-Allow-Origin": "*"
    },
    "bodyFileName": "chat-completion-response.json",
    "transformers": ["response-template"]
  }
}' --from-literal=embeddings.json='
{
  "request": {
    "method": "POST",
    "urlPath": "/v1/embeddings",
    "headers": {
      "Content-Type": {
        "contains": "application/json"
      }
    }
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json",
      "X-Request-ID": "{{randomValue type=\"UUID\"}}",
      "Access-Control-Allow-Origin": "*"
    },
    "bodyFileName": "embeddings-response.json",
    "transformers": ["response-template"]
  }
}' --from-literal=models.json='
{
  "request": {
    "method": "GET",
    "urlPath": "/v1/models"
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "bodyFileName": "models-response.json"
  }
}' --dry-run=client -o yaml | kubectl apply -f -
```

### Create Mock Response Templates
Create realistic response templates:

```bash
kubectl create configmap mock-openai-responses -n agentgateway-system --from-literal=chat-completion-response.json='
{
  "id": "chatcmpl-{{randomValue type=\"UUID\"}}",
  "object": "chat.completion",
  "created": {{now}},
  "model": "{{jsonPath request.body \"$.model\"}}",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "This is a mock response from the {{jsonPath request.body \"$.model\"}} model. The request was: {{jsonPath request.body \"$.messages[0].content\"}}. In a real scenario, this would be an AI-generated response with relevant information, insights, and helpful content tailored to your specific query."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": {{randomInt lower=10 upper=100}},
    "completion_tokens": {{randomInt lower=20 upper=150}},
    "total_tokens": {{randomInt lower=50 upper=200}}
  },
  "system_fingerprint": "fp_mock_{{randomValue type=\"ALPHANUMERIC\" length=10}}"
}' --from-literal=embeddings-response.json='
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [
        {{#repeat 1536}}{{randomDecimal lower=-1.0 upper=1.0}}{{#unless @last}},{{/unless}}{{/repeat}}
      ]
    }
  ],
  "model": "text-embedding-3-small",
  "usage": {
    "prompt_tokens": {{randomInt lower=1 upper=10}},
    "total_tokens": {{randomInt lower=1 upper=10}}
  }
}' --from-literal=models-response.json='
{
  "object": "list",
  "data": [
    {
      "id": "gpt-4o-mini",
      "object": "model",
      "created": 1677610602,
      "owned_by": "openai"
    },
    {
      "id": "gpt-4o",
      "object": "model",
      "created": 1677610602,
      "owned_by": "openai"
    },
    {
      "id": "text-embedding-3-small",
      "object": "model",
      "created": 1677610602,
      "owned_by": "openai"
    }
  ]
}' --dry-run=client -o yaml | kubectl apply -f -
```

## Configure AgentGateway for Mock OpenAI

### Create Mock OpenAI Backend
```bash
kubectl apply -f- <<'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mock-openai
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai:
        model: "mock-gpt-4o"
      host: mock-openai.agentgateway-system.svc.cluster.local
      port: 443
      path: "/v1/chat/completions"
  policies:
    auth:
      passthrough: {}
EOF
```

### Create Gateway and Route to Mock OpenAI
```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway  # Open source gateway class
  listeners:
  - name: http
    port: 8080
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mock-openai
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway
    namespace: agentgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /openai
    backendRefs:
    - name: mock-openai
      group: agentgateway.dev
      kind: AgentgatewayBackend
    timeouts:
      request: "120s"
EOF
```

## Testing the Mock Environment

### Verify Mock Server is Running
```bash
# Check mock OpenAI pods
kubectl get pods -n agentgateway-system -l app=mock-openai

# Check service
kubectl get svc mock-openai -n agentgateway-system

# Test mock server directly
kubectl port-forward -n agentgateway-system svc/mock-openai 8081:443 &
curl -s http://localhost:8081/v1/models | jq .
kill %1
```

### Test Through AgentGateway

#### Setup Access to Gateway (Kind Cluster)
Since we're using kind, we need to port-forward to access the gateway:

```bash
# Port-forward AgentGateway service in background
kubectl port-forward -n agentgateway-system svc/agentgateway 8080:8080 &

# Store the process ID to kill it later
PORTFORWARD_PID=$!
echo "Port-forward running as PID: $PORTFORWARD_PID"

# Set gateway endpoint
export GATEWAY_IP="localhost"
```

#### Test Chat Completions
```bash
# Test chat completion
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "mock-gpt-4o",
    "messages": [
      {
        "role": "user",
        "content": "Hello! Can you explain what a mock server is?"
      }
    ],
    "max_tokens": 100
  }'
```

Expected response:
```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1701234567,
  "model": "gpt-4o-mini",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "This is a mock response from the gpt-4o-mini model. The request was: Hello! Can you explain what a mock server is?..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 25,
    "completion_tokens": 87,
    "total_tokens": 112
  }
}
```

#### Test Embeddings
```bash
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "input": "Sample text for embedding",
    "model": "text-embedding-3-small"
  }'
```

#### Test Models List
```bash
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "mock-gpt-4o"
  }'
```

#### Stop Port-Forward
When done testing, stop the port-forward:
```bash
# Kill the port-forward process
kill $PORTFORWARD_PID
```

## Advanced Mock Configuration

### Simulate Different Response Scenarios
Create additional mappings for testing edge cases:

```bash
kubectl create configmap mock-openai-advanced -n agentgateway-system --from-literal=rate-limit.json='
{
  "request": {
    "method": "POST",
    "urlPath": "/v1/chat/completions",
    "headers": {
      "X-Test-Scenario": {
        "equalTo": "rate-limit"
      }
    }
  },
  "response": {
    "status": 429,
    "headers": {
      "Content-Type": "application/json",
      "Retry-After": "60"
    },
    "body": "{\"error\":{\"message\":\"Rate limit exceeded\",\"type\":\"rate_limit_error\"}}"
  }
}' --from-literal=error-scenario.json='
{
  "request": {
    "method": "POST",
    "urlPath": "/v1/chat/completions",
    "headers": {
      "X-Test-Scenario": {
        "equalTo": "error"
      }
    }
  },
  "response": {
    "status": 500,
    "headers": {
      "Content-Type": "application/json"
    },
    "body": "{\"error\":{\"message\":\"Internal server error\",\"type\":\"internal_error\"}}"
  }
}' --dry-run=client -o yaml | kubectl apply -f -

# Update the mock deployment to include advanced scenarios
kubectl patch deployment mock-openai -n agentgateway-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {
      "name": "mock-advanced",
      "configMap": {
        "name": "mock-openai-advanced"
      }
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/-",
    "value": {
      "name": "mock-advanced",
      "mountPath": "/home/wiremock/mappings/advanced"
    }
  }
]'
```

### Test Error Scenarios
```bash
# Test rate limiting
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "x-test-scenario: rate-limit" \
  -d '{
    "model": "mock-gpt-4o",
    "messages": [{"role": "user", "content": "Test rate limit"}]
  }'

# Test error scenario
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "x-test-scenario: error" \
  -d '{
    "model": "mock-gpt-4o", 
    "messages": [{"role": "user", "content": "Test error"}]
  }'
```

## Observing Mock Traffic

### View Gateway Logs
```bash
# AgentGateway logs
kubectl logs -n agentgateway-system deploy/agentgateway --tail=20

# Mock server logs
kubectl logs -n agentgateway-system deploy/mock-openai --tail=20

# Watch for AI-specific telemetry
kubectl logs -n agentgateway-system deploy/agentgateway -f | grep gen_ai
```

Gateway logs include AI-specific telemetry:
```
gen_ai.operation.name=chat
gen_ai.provider.name=openai
gen_ai.request.model=gpt-4o-mini
gen_ai.usage.input_tokens=15
gen_ai.usage.output_tokens=42
duration=250ms
```

## Load Testing with Mock Server

### Create Load Test Script
```bash
cat <<'EOF' > load-test-mock.sh
#!/bin/bash

GATEWAY_IP="${GATEWAY_IP:-localhost}"
GATEWAY_PORT="${GATEWAY_PORT:-8080}"
CONCURRENT_REQUESTS=${1:-10}
TOTAL_REQUESTS=${2:-100}

echo "Running load test against mock OpenAI..."
echo "Concurrent requests: $CONCURRENT_REQUESTS"
echo "Total requests: $TOTAL_REQUESTS"
echo "Target: $GATEWAY_IP:$GATEWAY_PORT"

# Function to send a request
send_request() {
  local id=$1
  curl -s -o /dev/null -w "Request $id: %{http_code} - %{time_total}s\n" \
    "$GATEWAY_IP:$GATEWAY_PORT/openai" \
    -H "content-type: application/json" \
    -d '{
      "model": "mock-gpt-4o",
      "messages": [
        {
          "role": "user",
          "content": "Load test request '"'"$id"'"' - tell me about AI gateways"
        }
      ]
    }'
}

# Send requests concurrently
echo "Starting load test..."
for ((i=1; i<=TOTAL_REQUESTS; i++)); do
  send_request $i &

  # Limit concurrent requests
  if ((i % CONCURRENT_REQUESTS == 0)); then
    wait
  fi
done

wait
echo "Load test complete!"
EOF

chmod +x load-test-mock.sh

# Run a small load test
./load-test-mock.sh 5 20
```

### Monitor Load Test Results
During the load test, observe:

- **Resource Usage**: `kubectl top pods -n agentgateway-system`
- **Gateway Logs**: `kubectl logs -n agentgateway-system deploy/agentgateway -f`
- **Request Patterns**: Look for AI telemetry in logs

## Development Workflow with Mock

### Typical Development Process

1. **Develop Against Mock**: Test routing, security, and features
2. **Validate Configuration**: Ensure routes and backends work correctly
3. **Load Test**: Verify performance characteristics
4. **Migrate to Real Providers**: Switch to production AI APIs

### Mock Configuration Management
Create a script to manage mock configurations:

```bash
cat <<'EOF' > mock-management.sh
#!/bin/bash

case $1 in
  "start")
    echo "Starting mock OpenAI server..."
    kubectl apply -f mock-openai-deployment.yaml
    kubectl wait --for=condition=available deployment/mock-openai -n agentgateway-system
    ;;
  "stop")
    echo "Stopping mock OpenAI server..."
    kubectl delete deployment mock-openai -n agentgateway-system
    kubectl delete svc mock-openai -n agentgateway-system
    ;;
  "restart")
    kubectl rollout restart deployment/mock-openai -n agentgateway-system
    ;;
  "status")
    kubectl get pods -n agentgateway-system -l app=mock-openai
    ;;
  "logs")
    kubectl logs deploy/mock-openai -n agentgateway-system -f
    ;;
  *)
    echo "Usage: $0 {start|stop|restart|status|logs}"
    ;;
esac
EOF

chmod +x mock-management.sh
```

## Troubleshooting Mock Setup

### Common Issues

1. **Mock Server Not Responding**
```bash
# Check pod status
kubectl get pods -n agentgateway-system -l app=mock-openai

# Check pod logs
kubectl logs deploy/mock-openai -n agentgateway-system

# Test connectivity
kubectl exec -n agentgateway-system deploy/agentgateway -- \
  curl -v http://mock-openai:443/v1/models
```

2. **Routes Not Working**
```bash
# Check HTTPRoute status
kubectl get httproute mock-openai -n agentgateway-system -o yaml

# Verify backend configuration
kubectl get agentgatewaybackend mock-openai -n agentgateway-system -o yaml

# Check gateway status
kubectl get gateway agentgateway -n agentgateway-system -o yaml
```

3. **Wrong Response Format**
```bash
# Check WireMock mappings
kubectl get configmap mock-openai-mappings -n agentgateway-system -o yaml

# Restart mock server to reload config
kubectl rollout restart deployment/mock-openai -n agentgateway-system
```

### Debugging Mock Responses
```bash
# Enable verbose logging in WireMock
kubectl patch deployment mock-openai -n agentgateway-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--global-response-templating"
  }
]'

# Check detailed logs
kubectl logs deploy/mock-openai -n agentgateway-system | grep -E "(Request|Response|Error)"
```

## Cleanup

When you're ready to move to real providers:

```bash
# Remove mock resources
kubectl delete httproute mock-openai -n agentgateway-system
kubectl delete agentgatewaybackend mock-openai -n agentgateway-system
kubectl delete deployment mock-openai -n agentgateway-system
kubectl delete svc mock-openai -n agentgateway-system
kubectl delete configmap mock-openai-mappings -n agentgateway-system
kubectl delete configmap mock-openai-responses -n agentgateway-system
kubectl delete configmap mock-openai-advanced -n agentgateway-system

# Clean up scripts
rm -f load-test-mock.sh mock-management.sh
```

## Next Steps

With your mock OpenAI environment working, you're ready to:

- **Test real AI providers** - Configure actual OpenAI, Anthropic, or other APIs
- **Implement security features** - Add rate limiting, PII protection, and prompt guards
- **Explore advanced routing** - Path-based, header-based, and intelligent routing
- **Scale testing** - Use mock for large-scale performance testing

In our next blog post, we'll connect to the real OpenAI API and compare the behavior with our mock environment, demonstrating how to seamlessly transition from development to production.

## Key Takeaways

- **Mock servers enable cost-effective development and testing**
- **Open source AgentGateway provides the same core functionality as Enterprise**
- **Realistic responses help validate routing and configuration logic**
- **Error simulation allows testing of failure scenarios**
- **Load testing is safer and cheaper with mock endpoints**
- **Deterministic behavior makes debugging and testing easier**
- **Seamless transition from mock to real providers maintains development velocity**

Your mock OpenAI environment provides a solid foundation for developing and testing AgentGateway configurations before connecting to production AI services!