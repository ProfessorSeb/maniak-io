---
title: "Creating a Mock OpenAI Development Environment with AgentGateway"
date: 2026-02-09
description: "Learn how to set up a cost-effective mock OpenAI server for development and testing with AgentGateway. Build realistic AI responses without API costs while developing routing patterns, observability features, and security configurations."
---

# Creating a Mock OpenAI Development Environment

## Introduction

Before connecting to real AI providers and spending money on API calls, it's valuable to have a local mock server that simulates OpenAI's API. This allows you to test your AgentGateway configuration, observability setup, routing patterns, and security features without incurring costs or rate limits.

In this guide, we'll deploy a mock OpenAI server that responds with realistic-looking AI responses, complete with token usage metrics and streaming support. This is perfect for development, testing, and demonstration environments.

## What You'll Learn

- Deploy a mock OpenAI API server in your kind cluster
- Configure AgentGateway to route to the mock server
- Test various OpenAI API endpoints (chat completions, embeddings)
- Understand request/response patterns before using real providers
- Use mock environments for load testing and development

## Prerequisites

- Completed previous blog posts: AgentGateway setup and observability stack
- Kind cluster with AgentGateway and monitoring running
- kubectl and basic understanding of Kubernetes concepts

## Understanding Mock OpenAI Benefits

### Why Use a Mock Server?

1. **Cost Control**: No API charges during development and testing
2. **Deterministic Responses**: Predictable outputs for testing
3. **Rate Limit Testing**: Simulate rate limiting without hitting real limits
4. **Offline Development**: Work without internet connectivity to AI providers
5. **Load Testing**: Test high traffic patterns safely
6. **Feature Development**: Test new features before production

## Environment Setup

Ensure your environment variables are set:

```bash
# Verify AgentGateway environment
export SOLO_TRIAL_LICENSE_KEY="your-license-key-here"
export ENTERPRISE_AGW_VERSION="2.1.0"

# Check cluster status
kubectl get pods -n enterprise-agentgateway
kubectl get pods -n monitoring
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
  namespace: enterprise-agentgateway
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
  namespace: enterprise-agentgateway
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
kubectl create configmap mock-openai-mappings -n enterprise-agentgateway --from-literal=chat-completions.json='
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
kubectl create configmap mock-openai-responses -n enterprise-agentgateway --from-literal=chat-completion-response.json='
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
  namespace: enterprise-agentgateway
spec:
  ai:
    provider:
      openai: 
        # Override the default OpenAI endpoint to use our mock
        config:
          endpoint: "https://mock-openai.enterprise-agentgateway.svc.cluster.local"
  static:
    host: mock-openai.enterprise-agentgateway.svc.cluster.local
    port: 443
  # Mock server doesn't need real authentication
  policies:
    auth:
      inline:
        Authorization: "Bearer mock-api-key"
    # Skip TLS verification since mock uses HTTP
    tls:
      skipVerify: true
EOF
```

### Create Route to Mock OpenAI

```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mock-openai
  namespace: enterprise-agentgateway
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /mock-openai
      backendRefs:
        - name: mock-openai
          group: agentgateway.dev
          kind: AgentgatewayBackend
      timeouts:
        request: "60s"
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1
EOF
```

## Testing the Mock Environment

### Verify Mock Server is Running

```bash
# Check mock OpenAI pods
kubectl get pods -n enterprise-agentgateway -l app=mock-openai

# Check service
kubectl get svc mock-openai -n enterprise-agentgateway

# Test mock server directly
kubectl port-forward -n enterprise-agentgateway svc/mock-openai 8081:443 &
curl -s http://localhost:8081/v1/models | jq .
kill %1
```

### Test Through AgentGateway

#### Test Chat Completions

```bash
# Get AgentGateway endpoint
export GATEWAY_IP=$(kubectl get svc -n enterprise-agentgateway --selector=gateway.networking.k8s.io/gateway-name=agentgateway -o jsonpath='{.items[*].status.loadBalancer.ingress[0].ip}{.items[*].status.loadBalancer.ingress[0].hostname}')

# For kind clusters, use localhost
if [ -z "$GATEWAY_IP" ]; then
  export GATEWAY_IP="localhost"
fi

# Test chat completion
curl -i "$GATEWAY_IP:8080/mock-openai/chat/completions" \
  -H "content-type: application/json" \
  -H "authorization: bearer mock-token" \
  -d '{
    "model": "gpt-4o-mini",
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
curl -i "$GATEWAY_IP:8080/mock-openai/embeddings" \
  -H "content-type: application/json" \
  -H "authorization: bearer mock-token" \
  -d '{
    "input": "Sample text for embedding",
    "model": "text-embedding-3-small"
  }'
```

#### Test Models List

```bash
curl -i "$GATEWAY_IP:8080/mock-openai/models" \
  -H "authorization: bearer mock-token"
```

## Advanced Mock Configuration

### Simulate Different Response Scenarios

Create additional mappings for testing edge cases:

```bash
kubectl create configmap mock-openai-advanced -n enterprise-agentgateway --from-literal=rate-limit.json='
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
kubectl patch deployment mock-openai -n enterprise-agentgateway --type='json' -p='[
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
curl -i "$GATEWAY_IP:8080/mock-openai/chat/completions" \
  -H "content-type: application/json" \
  -H "authorization: bearer mock-token" \
  -H "x-test-scenario: rate-limit" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Test rate limit"}]
  }'

# Test error scenario
curl -i "$GATEWAY_IP:8080/mock-openai/chat/completions" \
  -H "content-type: application/json" \
  -H "authorization: bearer mock-token" \
  -H "x-test-scenario: error" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Test error"}]
  }'
```

## Observing Mock Traffic

### View Metrics in Grafana

1. **Open Grafana**: `kubectl port-forward -n monitoring svc/grafana-prometheus 3000:3000`
2. **Navigate to AgentGateway Dashboard**
3. **Send test requests** and watch metrics populate:
   - Request rates
   - Token usage (from mock responses)
   - Response times
   - Error rates

### Check Traces

1. **Navigate to Explore** in Grafana
2. **Select Tempo** data source
3. **Search for traces** with service `agentgateway`
4. **Examine spans** to see:
   - Request processing time
   - Backend communication
   - Mock server response time

### View Logs

```bash
# AgentGateway logs
kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail=20

# Mock server logs
kubectl logs deploy/mock-openai -n enterprise-agentgateway --tail=20
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
    "$GATEWAY_IP:$GATEWAY_PORT/mock-openai/chat/completions" \
    -H "content-type: application/json" \
    -H "authorization: bearer mock-token" \
    -d '{
      "model": "gpt-4o-mini",
      "messages": [
        {
          "role": "user",
          "content": "Load test request '$id' - tell me about AI gateways"
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

1. **Grafana Dashboard**: Watch request rates spike
2. **Resource Usage**: `kubectl top pods -n enterprise-agentgateway`
3. **Logs**: `kubectl logs deploy/agentgateway -n enterprise-agentgateway -f`

## Development Workflow with Mock

### Typical Development Process

1. **Develop Against Mock**: Test routing, security, and features
2. **Validate Observability**: Ensure metrics and traces work correctly
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
    kubectl wait --for=condition=available deployment/mock-openai -n enterprise-agentgateway
    ;;
  "stop")
    echo "Stopping mock OpenAI server..."
    kubectl delete deployment mock-openai -n enterprise-agentgateway
    kubectl delete svc mock-openai -n enterprise-agentgateway
    ;;
  "restart")
    kubectl rollout restart deployment/mock-openai -n enterprise-agentgateway
    ;;
  "status")
    kubectl get pods -n enterprise-agentgateway -l app=mock-openai
    ;;
  "logs")
    kubectl logs deploy/mock-openai -n enterprise-agentgateway -f
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

**1. Mock Server Not Responding**
```bash
# Check pod status
kubectl get pods -n enterprise-agentgateway -l app=mock-openai

# Check pod logs
kubectl logs deploy/mock-openai -n enterprise-agentgateway

# Test connectivity
kubectl exec -n enterprise-agentgateway deploy/agentgateway -- \
  curl -v http://mock-openai:443/v1/models
```

**2. Routes Not Working**
```bash
# Check HTTPRoute status
kubectl get httproute mock-openai -n enterprise-agentgateway -o yaml

# Verify backend configuration
kubectl get agentgatewaybackend mock-openai -n enterprise-agentgateway -o yaml
```

**3. Wrong Response Format**
```bash
# Check WireMock mappings
kubectl get configmap mock-openai-mappings -n enterprise-agentgateway -o yaml

# Restart mock server to reload config
kubectl rollout restart deployment/mock-openai -n enterprise-agentgateway
```

### Debugging Mock Responses

```bash
# Enable verbose logging in WireMock
kubectl patch deployment mock-openai -n enterprise-agentgateway --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--global-response-templating"
  }
]'

# Check detailed logs
kubectl logs deploy/mock-openai -n enterprise-agentgateway | grep -E "(Request|Response|Error)"
```

## Cleanup

When you're ready to move to real providers:

```bash
# Remove mock resources
kubectl delete httproute mock-openai -n enterprise-agentgateway
kubectl delete agentgatewaybackend mock-openai -n enterprise-agentgateway
kubectl delete deployment mock-openai -n enterprise-agentgateway
kubectl delete svc mock-openai -n enterprise-agentgateway
kubectl delete configmap mock-openai-mappings -n enterprise-agentgateway
kubectl delete configmap mock-openai-responses -n enterprise-agentgateway
kubectl delete configmap mock-openai-advanced -n enterprise-agentgateway

# Clean up scripts
rm -f load-test-mock.sh mock-management.sh
```

## Next Steps

With your mock OpenAI environment working, you're ready to:

1. **Test real AI providers** - Configure actual OpenAI, Anthropic, or other APIs
2. **Implement security features** - Add authentication, rate limiting, and guardrails
3. **Explore advanced routing** - Path-based, header-based, and intelligent routing
4. **Scale testing** - Use mock for large-scale performance testing

In our next blog post, we'll connect to the real OpenAI API and compare the behavior with our mock environment, demonstrating how to seamlessly transition from development to production.

## Key Takeaways

- **Mock servers** enable cost-effective development and testing
- **Realistic responses** help validate observability and routing logic
- **Error simulation** allows testing of failure scenarios  
- **Load testing** is safer and cheaper with mock endpoints
- **Deterministic behavior** makes debugging and testing easier
- **Seamless transition** from mock to real providers maintains development velocity

Your mock OpenAI environment provides a solid foundation for developing and testing AgentGateway configurations before connecting to production AI services!