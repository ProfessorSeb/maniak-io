---
title: "Advanced Routing Patterns for AI Models with AgentGateway"
date: 2026-02-09
description: "Master sophisticated AI traffic management with AgentGateway's advanced routing patterns. Learn path-based, header-based, weighted, and content-based routing for intelligent model selection, A/B testing, cost optimization, and failover strategies."
---

# Advanced Routing Patterns for AI Models

## Introduction

One of the most powerful features of AgentGateway is its ability to intelligently route requests to different AI models or providers based on various criteria. This enables sophisticated scenarios like model selection based on request characteristics, A/B testing different models, and cost optimization through intelligent routing.

In this guide, we'll explore multiple routing patterns that transform your AgentGateway from a simple proxy into an intelligent AI traffic management system.

## What You'll Learn

- Path-based routing for different models and use cases
- Header-based routing for tenant isolation and testing
- Query parameter routing for flexible client control
- Weighted routing for A/B testing and gradual rollouts
- Content-based routing using request body analysis
- Fallback and failover routing strategies
- Cost-optimized routing patterns

## Prerequisites

- Completed previous blog posts (AgentGateway setup, observability, OpenAI integration)
- Understanding of Kubernetes Gateway API concepts
- Knowledge of HTTP routing principles
- OpenAI API key for testing (we'll add Anthropic optionally)

## Environment Setup

Ensure your environment is ready:

```bash
# Verify environment variables
export OPENAI_API_KEY="sk-your-openai-api-key-here"
export SOLO_TRIAL_LICENSE_KEY="your-license-key-here"

# Verify AgentGateway is running
kubectl get pods -n enterprise-agentgateway

# Verify existing OpenAI configuration
kubectl get agentgatewaybackend openai-all-models -n enterprise-agentgateway
```

## Pattern 1: Path-Based Routing

### Model-Specific Paths

Create different paths for different models, allowing clients to choose the right model for their use case:

```bash
kubectl apply -f- <<'EOF'
# GPT-4o Mini route (fast, cost-effective)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-gpt4o-mini
  namespace: enterprise-agentgateway
  labels:
    provider: openai
    model: gpt-4o-mini
    use-case: general
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /ai/fast
        - path:
            type: PathPrefix
            value: /ai/gpt4o-mini
      
      backendRefs:
        - name: openai-gpt4o-mini
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      timeouts:
        request: "60s"
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        # Add model override header
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Model-Used
                value: gpt-4o-mini
              - name: X-Use-Case
                value: fast-general

---
# GPT-4o route (premium quality)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-gpt4o
  namespace: enterprise-agentgateway
  labels:
    provider: openai
    model: gpt-4o
    use-case: premium
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /ai/premium
        - path:
            type: PathPrefix
            value: /ai/gpt4o
      
      backendRefs:
        - name: openai-gpt4o
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
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Model-Used
                value: gpt-4o
              - name: X-Use-Case
                value: premium-quality

---
# Embeddings route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-embeddings-route
  namespace: enterprise-agentgateway
  labels:
    provider: openai
    endpoint: embeddings
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /ai/embeddings
      
      backendRefs:
        - name: openai-all-models
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/embeddings
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Service-Type
                value: embeddings
EOF
```

### Create Model-Specific Backends

```bash
kubectl apply -f- <<'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-gpt4o-mini
  namespace: enterprise-agentgateway
spec:
  ai:
    provider:
      openai:
        config:
          model: "gpt-4o-mini"  # Force specific model
  policies:
    auth:
      secretRef:
        name: openai-secret
    timeout:
      request: "60s"

---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-gpt4o
  namespace: enterprise-agentgateway
spec:
  ai:
    provider:
      openai:
        config:
          model: "gpt-4o"  # Force premium model
  policies:
    auth:
      secretRef:
        name: openai-secret
    timeout:
      request: "120s"
EOF
```

### Test Path-Based Routing

```bash
export GATEWAY_IP="${GATEWAY_IP:-localhost}"
export GATEWAY_PORT="${GATEWAY_PORT:-8080}"

# Test fast/general route (gpt-4o-mini)
echo "=== Testing Fast Route (GPT-4o Mini) ==="
curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/fast/completions" \
  -H "content-type: application/json" \
  -d '{
    "messages": [
      {
        "role": "user",
        "content": "What model are you and what are your strengths?"
      }
    ],
    "max_tokens": 100
  }' | jq '{
    model: .model,
    content: .choices[0].message.content,
    usage: .usage
  }'

echo ""
echo "=== Testing Premium Route (GPT-4o) ==="
curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/premium/completions" \
  -H "content-type: application/json" \
  -d '{
    "messages": [
      {
        "role": "user", 
        "content": "What model are you and what are your strengths?"
      }
    ],
    "max_tokens": 100
  }' | jq '{
    model: .model,
    content: .choices[0].message.content,
    usage: .usage
  }'
```

## Pattern 2: Header-Based Routing

### Tenant Isolation and Testing

Use headers to route requests to different models or providers based on client characteristics:

```bash
kubectl apply -f- <<'EOF'
# Development environment route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-dev-environment
  namespace: enterprise-agentgateway
  labels:
    environment: development
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /ai/chat
          headers:
            - name: X-Environment
              value: development
        - path:
            type: PathPrefix
            value: /ai/chat
          headers:
            - name: X-Environment
              value: dev
      
      backendRefs:
        - name: openai-gpt4o-mini  # Use cheaper model for dev
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Environment-Used
                value: development
              - name: X-Model-Tier
                value: economy

---
# Production environment route  
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-prod-environment
  namespace: enterprise-agentgateway
  labels:
    environment: production
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /ai/chat
          headers:
            - name: X-Environment
              value: production
        - path:
            type: PathPrefix
            value: /ai/chat
          headers:
            - name: X-Environment
              value: prod
      
      backendRefs:
        - name: openai-gpt4o  # Use premium model for prod
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Environment-Used
                value: production
              - name: X-Model-Tier
                value: premium

---
# A/B Testing route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-ab-test
  namespace: enterprise-agentgateway
  labels:
    purpose: ab-testing
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    # Variant A - GPT-4o Mini
    - matches:
        - path:
            type: PathPrefix
            value: /ai/chat
          headers:
            - name: X-AB-Test
              value: variant-a
      
      backendRefs:
        - name: openai-gpt4o-mini
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-AB-Variant
                value: variant-a
              - name: X-Model-Used
                value: gpt-4o-mini
    
    # Variant B - GPT-4o
    - matches:
        - path:
            type: PathPrefix
            value: /ai/chat
          headers:
            - name: X-AB-Test
              value: variant-b
      
      backendRefs:
        - name: openai-gpt4o
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-AB-Variant
                value: variant-b
              - name: X-Model-Used
                value: gpt-4o
EOF
```

### Test Header-Based Routing

```bash
# Test development environment routing
echo "=== Testing Development Environment ==="
curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/chat/completions" \
  -H "content-type: application/json" \
  -H "X-Environment: development" \
  -d '{
    "messages": [
      {"role": "user", "content": "Hello from development!"}
    ],
    "max_tokens": 50
  }' | jq '{
    model: .model,
    content: .choices[0].message.content
  }'

echo ""
echo "=== Testing Production Environment ==="
curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/chat/completions" \
  -H "content-type: application/json" \
  -H "X-Environment: production" \
  -d '{
    "messages": [
      {"role": "user", "content": "Hello from production!"}
    ],
    "max_tokens": 50
  }' | jq '{
    model: .model,
    content: .choices[0].message.content
  }'

echo ""
echo "=== Testing A/B Test Variant A ==="
curl -i "$GATEWAY_IP:$GATEWAY_PORT/ai/chat/completions" \
  -H "content-type: application/json" \
  -H "X-AB-Test: variant-a" \
  -d '{
    "messages": [
      {"role": "user", "content": "A/B test message"}
    ],
    "max_tokens": 30
  }' | grep -E "(X-AB-Variant|X-Model-Used)"

echo ""
echo "=== Testing A/B Test Variant B ==="
curl -i "$GATEWAY_IP:$GATEWAY_PORT/ai/chat/completions" \
  -H "content-type: application/json" \
  -H "X-AB-Test: variant-b" \
  -d '{
    "messages": [
      {"role": "user", "content": "A/B test message"}
    ],
    "max_tokens": 30
  }' | grep -E "(X-AB-Variant|X-Model-Used)"
```

## Pattern 3: Query Parameter Routing

### Flexible Client-Side Model Selection

Allow clients to specify routing preferences via query parameters:

```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-query-param-routing
  namespace: enterprise-agentgateway
  labels:
    routing-type: query-parameter
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    # Route for speed preference
    - matches:
        - path:
            type: PathPrefix
            value: /ai/flexible
          queryParams:
            - name: speed
              value: fast
      
      backendRefs:
        - name: openai-gpt4o-mini
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Speed-Optimized
                value: "true"
    
    # Route for quality preference
    - matches:
        - path:
            type: PathPrefix
            value: /ai/flexible
          queryParams:
            - name: quality
              value: premium
      
      backendRefs:
        - name: openai-gpt4o
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Quality-Optimized
                value: "true"
    
    # Route for cost preference
    - matches:
        - path:
            type: PathPrefix
            value: /ai/flexible
          queryParams:
            - name: cost
              value: minimal
      
      backendRefs:
        - name: openai-gpt4o-mini
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Cost-Optimized
                value: "true"
EOF
```

### Test Query Parameter Routing

```bash
# Test speed-optimized routing
echo "=== Testing Speed-Optimized Routing ==="
curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/flexible/completions?speed=fast" \
  -H "content-type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "Fast response needed!"}
    ],
    "max_tokens": 50
  }' | jq '{model: .model, usage: .usage}'

# Test quality-optimized routing  
echo "=== Testing Quality-Optimized Routing ==="
curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/flexible/completions?quality=premium" \
  -H "content-type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "High quality response needed!"}
    ],
    "max_tokens": 50
  }' | jq '{model: .model, usage: .usage}'

# Test cost-optimized routing
echo "=== Testing Cost-Optimized Routing ==="
curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/flexible/completions?cost=minimal" \
  -H "content-type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "Cost-effective response needed!"}
    ],
    "max_tokens": 50
  }' | jq '{model: .model, usage: .usage}'
```

## Pattern 4: Weighted Routing for Gradual Rollouts

### Implement Traffic Splitting

Use multiple backends with different weights for gradual model rollouts:

```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-weighted-routing
  namespace: enterprise-agentgateway
  labels:
    routing-type: weighted
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /ai/gradual-rollout
      
      # Split traffic between models
      backendRefs:
        # 80% traffic to stable model
        - name: openai-gpt4o-mini
          group: agentgateway.dev
          kind: AgentgatewayBackend
          weight: 80
        # 20% traffic to new model for testing
        - name: openai-gpt4o
          group: agentgateway.dev
          kind: AgentgatewayBackend
          weight: 20
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Routing-Type
                value: weighted-rollout
EOF
```

### Test Weighted Routing

```bash
cat <<'EOF' > test-weighted-routing.sh
#!/bin/bash

GATEWAY_IP="${GATEWAY_IP:-localhost}"
GATEWAY_PORT="${GATEWAY_PORT:-8080}"
REQUESTS=${1:-20}

echo "Testing weighted routing with $REQUESTS requests..."
echo "Expected: ~80% gpt-4o-mini, ~20% gpt-4o"
echo ""

declare -A model_counts

for ((i=1; i<=REQUESTS; i++)); do
    response=$(curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/gradual-rollout/completions" \
        -H "content-type: application/json" \
        -d '{
            "messages": [
                {"role": "user", "content": "What model are you?"}
            ],
            "max_tokens": 10
        }')
    
    model=$(echo "$response" | jq -r '.model')
    ((model_counts[$model]++))
    
    printf "Request %2d: %s\n" $i "$model"
done

echo ""
echo "=== Results ==="
for model in "${!model_counts[@]}"; do
    count=${model_counts[$model]}
    percentage=$(( count * 100 / REQUESTS ))
    echo "$model: $count requests (${percentage}%)"
done
EOF

chmod +x test-weighted-routing.sh
./test-weighted-routing.sh 10
```

## Pattern 5: Content-Based Routing

### Route Based on Request Content

Use AgentGateway's request analysis capabilities to route based on content characteristics:

```bash
kubectl apply -f- <<'EOF'
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: content-based-routing
  namespace: enterprise-agentgateway
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agentgateway
  traffic:
    # Simple content analysis - route long requests to premium model
    requestTransformation:
      inline:
        # Add request length as header for routing decisions
        headers:
          add:
            X-Content-Length: 'string(len(json(request.body).messages[0].content))'
            X-Content-Type: 'if len(json(request.body).messages[0].content) > 100 then "long" else "short"'

---
# Route short content to fast model
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-short-content
  namespace: enterprise-agentgateway
  labels:
    routing-type: content-based
    content-type: short
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /ai/smart
          headers:
            - name: X-Content-Type
              value: short
      
      backendRefs:
        - name: openai-gpt4o-mini
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Content-Routing
                value: short-content-fast-model

---
# Route long content to premium model  
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-long-content
  namespace: enterprise-agentgateway
  labels:
    routing-type: content-based
    content-type: long
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /ai/smart
          headers:
            - name: X-Content-Type
              value: long
      
      backendRefs:
        - name: openai-gpt4o
          group: agentgateway.dev
          kind: AgentgatewayBackend
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Content-Routing
                value: long-content-premium-model
EOF
```

### Test Content-Based Routing

```bash
# Test with short content
echo "=== Testing Short Content (should use gpt-4o-mini) ==="
curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/smart/completions" \
  -H "content-type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "Hi!"}
    ],
    "max_tokens": 20
  }' | jq '{model: .model, usage: .usage}'

echo ""
echo "=== Testing Long Content (should use gpt-4o) ==="
curl -s "$GATEWAY_IP:$GATEWAY_PORT/ai/smart/completions" \
  -H "content-type: application/json" \
  -d '{
    "messages": [
      {
        "role": "user", 
        "content": "This is a much longer prompt that contains significantly more content and should trigger the routing logic to use the premium model because it requires more sophisticated processing and understanding. The content-based routing should detect this as a long request and route it to GPT-4o for better handling of complex queries that require more nuanced responses and deeper analysis."
      }
    ],
    "max_tokens": 50
  }' | jq '{model: .model, usage: .usage}'
```

## Observing Routing Patterns

### Create Routing Dashboard

```bash
cat <<'EOF' > routing-analysis.sh
#!/bin/bash

echo "Analyzing routing patterns from AgentGateway logs..."
echo ""

# Extract routing information from recent logs
kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail=100 | \
  jq -r 'select(.gen_ai) | [
    .timestamp,
    .route_name // "unknown",
    .gen_ai.request.model,
    (.gen_ai.usage.total_tokens // 0),
    (.request.headers["x-environment"] // "none"),
    (.request.headers["x-ab-test"] // "none")
  ] | @csv' | \
  awk -F',' '
BEGIN {
    print "Route Analysis Report"
    print "===================="
    print ""
    total_requests = 0
    split("", route_counts)
    split("", model_counts)
    split("", env_counts)
}
{
    total_requests++
    
    # Remove quotes
    route = $2
    model = $3
    tokens = $4
    env = $5
    
    gsub(/"/, "", route)
    gsub(/"/, "", model) 
    gsub(/"/, "", env)
    
    route_counts[route]++
    model_counts[model]++
    if (env != "none") env_counts[env]++
    
    total_tokens += tokens
}
END {
    print "Total Requests: " total_requests
    print "Total Tokens: " total_tokens
    print ""
    
    print "Routes:"
    for (route in route_counts) {
        printf "  %s: %d requests\n", route, route_counts[route]
    }
    print ""
    
    print "Models:"
    for (model in model_counts) {
        printf "  %s: %d requests\n", model, model_counts[model]
    }
    print ""
    
    print "Environments:"
    for (env in env_counts) {
        printf "  %s: %d requests\n", env, env_counts[env]
    }
}'
EOF

chmod +x routing-analysis.sh
./routing-analysis.sh
```

### Monitor Routing in Grafana

1. **Open Grafana Dashboard**:
   ```bash
   kubectl port-forward -n monitoring svc/grafana-prometheus 3000:3000 &
   ```

2. **Create Custom Routing Queries**:
   ```promql
   # Request rate by route
   sum(rate(agentgateway_requests_total[5m])) by (route_name)
   
   # Request rate by model
   sum(rate(agentgateway_requests_total[5m])) by (model)
   
   # Token usage by routing pattern
   sum(rate(agentgateway_tokens_total[5m])) by (route_name, token_type)
   ```

3. **Watch routing patterns** as you send test requests through different patterns

## Pattern 6: Fallback and Failover Routing

### Primary-Secondary Backend Configuration

```bash
kubectl apply -f- <<'EOF'
# Primary backend with health checking
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-primary
  namespace: enterprise-agentgateway
spec:
  ai:
    provider:
      openai:
        config:
          model: "gpt-4o"
  policies:
    auth:
      secretRef:
        name: openai-secret
    # Health checking configuration
    healthCheck:
      interval: "30s"
      timeout: "5s"
      unhealthyThreshold: 2
      healthyThreshold: 2

---
# Fallback backend
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-fallback
  namespace: enterprise-agentgateway
spec:
  ai:
    provider:
      openai:
        config:
          model: "gpt-4o-mini"
  policies:
    auth:
      secretRef:
        name: openai-secret

---
# Failover route configuration
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-failover
  namespace: enterprise-agentgateway
  labels:
    routing-type: failover
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /ai/reliable
      
      backendRefs:
        # Primary backend
        - name: openai-primary
          group: agentgateway.dev
          kind: AgentgatewayBackend
          weight: 100
        # Fallback backend (only used if primary fails)
        - name: openai-fallback
          group: agentgateway.dev
          kind: AgentgatewayBackend
          weight: 0
      
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: X-Routing-Type
                value: failover-enabled
EOF
```

## Creating Routing Test Suite

### Comprehensive Routing Test

```bash
cat <<'EOF' > comprehensive-routing-test.sh
#!/bin/bash

GATEWAY_IP="${GATEWAY_IP:-localhost}"
GATEWAY_PORT="${GATEWAY_PORT:-8080}"

echo "Comprehensive Routing Pattern Test Suite"
echo "======================================="
echo ""

test_endpoint() {
    local name="$1"
    local endpoint="$2"
    local headers="$3"
    local expected_model="$4"
    
    echo "Testing: $name"
    echo "Endpoint: $endpoint"
    
    local response=$(curl -s $headers "$GATEWAY_IP:$GATEWAY_PORT$endpoint" \
        -H "content-type: application/json" \
        -d '{
            "messages": [
                {"role": "user", "content": "What model are you?"}
            ],
            "max_tokens": 20
        }')
    
    local actual_model=$(echo "$response" | jq -r '.model')
    local tokens=$(echo "$response" | jq -r '.usage.total_tokens')
    
    if [[ "$actual_model" == *"$expected_model"* ]]; then
        echo "✓ Success: $actual_model ($tokens tokens)"
    else
        echo "✗ Failed: Expected $expected_model, got $actual_model"
    fi
    echo ""
}

echo "1. Path-based routing tests"
echo "---------------------------"
test_endpoint "Fast Route" "/ai/fast/completions" "" "gpt-4o-mini"
test_endpoint "Premium Route" "/ai/premium/completions" "" "gpt-4o"

echo "2. Header-based routing tests"  
echo "-----------------------------"
test_endpoint "Dev Environment" "/ai/chat/completions" "-H 'X-Environment: development'" "gpt-4o-mini"
test_endpoint "Prod Environment" "/ai/chat/completions" "-H 'X-Environment: production'" "gpt-4o"
test_endpoint "A/B Test Variant A" "/ai/chat/completions" "-H 'X-AB-Test: variant-a'" "gpt-4o-mini"
test_endpoint "A/B Test Variant B" "/ai/chat/completions" "-H 'X-AB-Test: variant-b'" "gpt-4o"

echo "3. Query parameter routing tests"
echo "--------------------------------"
test_endpoint "Speed Optimized" "/ai/flexible/completions?speed=fast" "" "gpt-4o-mini"
test_endpoint "Quality Optimized" "/ai/flexible/completions?quality=premium" "" "gpt-4o"
test_endpoint "Cost Optimized" "/ai/flexible/completions?cost=minimal" "" "gpt-4o-mini"

echo "Routing test suite complete!"
echo ""
echo "Check Grafana dashboard to see routing patterns and metrics."
EOF

chmod +x comprehensive-routing-test.sh
./comprehensive-routing-test.sh
```

## Best Practices and Considerations

### Routing Strategy Guidelines

1. **Performance vs Cost**: Balance response quality with token costs
2. **Gradual Rollouts**: Use weighted routing for safe model updates
3. **Environment Separation**: Use headers for dev/staging/prod isolation
4. **Content Awareness**: Route based on request complexity
5. **Fallback Planning**: Always have backup routes for reliability

### Monitoring Routing Health

```bash
cat <<'EOF' > routing-health-check.sh
#!/bin/bash

echo "Routing Health Check"
echo "==================="
echo ""

# Check route statuses
echo "Route Status:"
kubectl get httproute -n enterprise-agentgateway -o custom-columns=\
"NAME:.metadata.name,\
ACCEPTED:.status.parents[0].conditions[?(@.type=='Accepted')].status,\
AGE:.metadata.creationTimestamp"

echo ""

# Check backend health
echo "Backend Status:"  
kubectl get agentgatewaybackend -n enterprise-agentgateway -o custom-columns=\
"NAME:.metadata.name,\
ACCEPTED:.status.conditions[?(@.type=='Accepted')].status,\
AGE:.metadata.creationTimestamp"

echo ""

# Test each route with a quick health check
echo "Route Connectivity Tests:"
routes=(
    "/ai/fast/completions"
    "/ai/premium/completions"  
    "/ai/flexible/completions?speed=fast"
    "/ai/gradual-rollout/completions"
)

for route in "${routes[@]}"; do
    printf "%-35s: " "$route"
    response_code=$(curl -s -o /dev/null -w "%{http_code}" \
        "${GATEWAY_IP:-localhost}:${GATEWAY_PORT:-8080}$route" \
        -H "content-type: application/json" \
        -d '{"messages":[{"role":"user","content":"test"}],"max_tokens":1}')
    
    if [ "$response_code" = "200" ]; then
        echo "✓ OK"
    else
        echo "✗ $response_code"
    fi
done
EOF

chmod +x routing-health-check.sh
./routing-health-check.sh
```

## Cleanup

When you want to clean up the routing configurations:

```bash
# Remove all routing test configurations
kubectl delete httproute -n enterprise-agentgateway -l routing-type
kubectl delete httproute -n enterprise-agentgateway \
  openai-gpt4o-mini openai-gpt4o openai-embeddings-route \
  openai-dev-environment openai-prod-environment openai-ab-test \
  openai-query-param-routing openai-weighted-routing \
  openai-short-content openai-long-content openai-failover

# Remove test backends (keep original openai-all-models)
kubectl delete agentgatewaybackend -n enterprise-agentgateway \
  openai-gpt4o-mini openai-gpt4o openai-primary openai-fallback

# Remove policy
kubectl delete enterpriseagentgatewaypolicy content-based-routing -n enterprise-agentgateway

# Clean up test scripts
rm -f test-weighted-routing.sh routing-analysis.sh comprehensive-routing-test.sh routing-health-check.sh
```

## Next Steps

With advanced routing patterns mastered, you're ready to:

1. **Add security layers** - Authentication, authorization, and rate limiting
2. **Implement guardrails** - Content filtering and safety policies
3. **Multi-provider routing** - Add Anthropic, AWS Bedrock, and others
4. **Production optimization** - Performance tuning and cost management

In our next blog post, we'll explore security features including JWT authentication, API key management, and role-based access control.

## Key Takeaways

- **Intelligent routing** transforms AgentGateway into a sophisticated AI traffic manager
- **Multiple routing criteria** enable complex decision-making logic  
- **A/B testing and gradual rollouts** provide safe model deployment strategies
- **Content-based routing** optimizes cost and performance automatically
- **Observability** provides insights into routing effectiveness and patterns
- **Fallback strategies** ensure reliability even when primary models fail

Your AgentGateway now has enterprise-grade routing capabilities that can handle complex production scenarios while optimizing for cost, performance, and reliability!