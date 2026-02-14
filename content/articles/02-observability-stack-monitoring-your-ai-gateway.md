---
title: "Observability Stack: Monitoring Your AI Gateway"
date: 2026-02-09
description: "Deploy a complete observability stack for AgentGateway including distributed tracing with Tempo, metrics with Prometheus and Grafana, and AI-specific monitoring dashboards for comprehensive visibility into performance and costs."
---


## Introduction

Observability is crucial for any production AI Gateway deployment. Enterprise AgentGateway emits comprehensive OpenTelemetry-compatible metrics, logs, and traces out of the box. In this guide, we'll deploy a complete observability stack including Grafana, Prometheus, Tempo, and Loki to collect, store, and visualize this rich telemetry data.

This setup will give you real-time visibility into your AI Gateway's performance, cost metrics, token usage, streaming performance, and more.

## What You'll Learn

- Deploy Tempo for distributed tracing
- Install Prometheus and Grafana for metrics and visualization
- Configure AgentGateway-specific monitoring
- Set up the official AgentGateway Grafana dashboard
- Access and interpret observability data

## Prerequisites

- Kind cluster with Enterprise AgentGateway installed (from Part 1)
- kubectl configured to work with your cluster
- Helm 3.x installed

## Architecture Overview

Our observability stack will include:

- **Grafana**: Visualization and dashboards
- **Prometheus**: Metrics collection and storage
- **Tempo**: Distributed tracing backend
- **Loki**: Log aggregation (optional)
- **AgentGateway Datasources**: Pre-configured Grafana dashboards

## Deploy Monitoring Stack

### Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

### Install Prometheus

First, let's deploy Prometheus to collect metrics:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus \
    --namespace monitoring \
    --set server.service.type=NodePort \
    --set server.service.nodePort=30090 \
    --set server.persistentVolume.enabled=false \
    --set alertmanager.enabled=false \
    --set kube-state-metrics.enabled=true \
    --set nodeExporter.enabled=false \
    --set pushgateway.enabled=false
```

### Install Tempo

Deploy Tempo for distributed tracing:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create Tempo configuration
cat <<EOF > tempo-values.yaml
tempo:
  retention: 24h
  receivers:
    jaeger:
      protocols:
        grpc:
          endpoint: 0.0.0.0:14250
        thrift_http:
          endpoint: 0.0.0.0:14268
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

service:
  type: NodePort
  
storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces

compactor:
  retention: 24h

metrics_generator:
  enabled: true
  registry:
    external_labels:
      source: tempo
EOF

helm install tempo grafana/tempo \
    --namespace monitoring \
    --values tempo-values.yaml
```

### Install Grafana

Deploy Grafana with pre-configured datasources:

```bash
# Create Grafana configuration
cat <<EOF > grafana-values.yaml
service:
  type: NodePort
  nodePort: 30080

persistence:
  enabled: false

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server:80
      access: proxy
      isDefault: true
    - name: Tempo
      type: tempo
      url: http://tempo:3100
      access: proxy

dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards/default

dashboards:
  default:
    agentgateway-genai:
      gnetId: 21703
      revision: 1
      datasource: Prometheus

adminPassword: admin
EOF

helm install grafana grafana/grafana \
    --namespace monitoring \
    --values grafana-values.yaml
```

## Configure AgentGateway for Observability

### Update AgentGateway Configuration

We need to configure AgentGateway to send traces to our Tempo instance:

```bash
kubectl apply -f- <<'EOF'
---
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayParameters
metadata:
  name: agentgateway-params
  namespace: enterprise-agentgateway
spec:
  # Enable shared extensions
  sharedExtensions:
    extauth:
      enabled: true
      deployment:
        spec:
          replicas: 1
    ratelimiter:
      enabled: true
      deployment:
        spec:
          replicas: 1
    extCache:
      enabled: true
      deployment:
        spec:
          replicas: 1
  
  # Enhanced observability configuration
  rawConfig:
    config:
      logging:
        level: info
        fields:
          add:
            jwt: 'jwt'
            request.body: json(request.body)
            response.body: json(response.body)
            request.body.modelId: json(request.body).modelId
            llm.provider: 'llm.provider'
            llm.model: 'llm.requestModel'
            llm.tokens.input: 'llm.inputTokens'
            llm.tokens.output: 'llm.outputTokens'
            llm.cost.total: 'llm.totalCost'
        format: json
        
      tracing:
        randomSampling: 'true'
        collector:
          endpoint: tempo.monitoring.svc.cluster.local:4317
        fields:
          add:
            # GenAI semantic conventions
            gen_ai.operation.name: '"chat"'
            gen_ai.system: "llm.provider"
            gen_ai.prompt: 'llm.prompt'
            gen_ai.completion: 'llm.completion.map(c, {"role":"assistant", "content": c})'
            gen_ai.request.model: "llm.requestModel"
            gen_ai.response.model: "llm.responseModel"
            gen_ai.usage.completion_tokens: "llm.outputTokens"
            gen_ai.usage.prompt_tokens: "llm.inputTokens"
            gen_ai.request: 'flatten(llm.params)'
            
            # Additional context
            jwt: 'jwt'
            response.body: 'json(response.body)'
            llm.cost.total: 'llm.totalCost'
            llm.cost.input: 'llm.inputCost'
            llm.cost.output: 'llm.outputCost'
            
      metrics:
        enabled: true
        prefix: "agentgateway"
        tags:
          add:
            provider: 'llm.provider'
            model: 'llm.requestModel'
            user: 'jwt.sub // "anonymous"'
            
  # Service configuration for monitoring
  service:
    spec:
      type: NodePort
      ports:
      - name: http
        port: 8080
        targetPort: 8080
        nodePort: 30080
        protocol: TCP
      - name: metrics
        port: 9091
        targetPort: 9091
        nodePort: 30091
        protocol: TCP
  
  # Deployment configuration
  deployment:
    spec:
      replicas: 1
      template:
        spec:
          containers:
          - name: agentgateway
            resources:
              requests:
                cpu: 200m
                memory: 128Mi
              limits:
                cpu: 500m
                memory: 256Mi
EOF
```

## Verify Monitoring Stack

### Check All Pods are Running

```bash
kubectl get pods -n monitoring
```

Expected output:
```bash
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-7b8c4c4d4c-xyz12               1/1     Running   0          3m
prometheus-server-5f8b8b7d7d-abc34     1/1     Running   0          5m
tempo-0                                1/1     Running   0          4m
```

### Verify AgentGateway Configuration

```bash
kubectl get pods -n enterprise-agentgateway
kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail=20
```

Look for log entries indicating successful startup and trace collection.

## Access Monitoring Interfaces

### Grafana Dashboard

Access Grafana at: `http://localhost:30080`
- Username: `admin`
- Password: `admin`

### Prometheus UI

Access Prometheus at: `http://localhost:30090`

### AgentGateway Metrics

Access AgentGateway metrics directly: `http://localhost:30091/metrics`

## AgentGateway GenAI Dashboard

The Grafana deployment automatically imports the official AgentGateway dashboard (ID: 21703). This dashboard provides:

### Key Metrics Panels

1. **Request Rate and Latency**
   - Requests per second by provider
   - P95, P99 latency percentiles
   - Error rates and status codes

2. **Token Usage and Costs**
   - Input/output token consumption
   - Cost tracking per provider
   - Token efficiency metrics

3. **Model Performance**
   - Response time by model
   - Token generation rates
   - Streaming performance

4. **Provider Health**
   - Provider availability
   - Error rates by provider
   - Failover events

### Using the Dashboard

1. Navigate to **Dashboards > Browse** in Grafana
2. Open **AgentGateway GenAI Dashboard**
3. Set time range (e.g., Last 1 hour)
4. Select providers/models using dropdown filters

## Testing Your Monitoring Setup

Let's create some test traffic to see data flowing through our observability stack:

### Deploy Test Mock Backend

```bash
kubectl apply -f- <<'EOF'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mock-openai
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mock-openai
  template:
    metadata:
      labels:
        app: mock-openai
    spec:
      containers:
      - name: wiremock
        image: wiremock/wiremock:3.3.1
        ports:
        - containerPort: 8080
        args:
        - --port=8080
        - --verbose
        volumeMounts:
        - name: wiremock-data
          mountPath: /home/wiremock
        env:
        - name: JAVA_OPTS
          value: "-Dfile.encoding=UTF-8"
      volumes:
      - name: wiremock-data
        configMap:
          name: mock-openai-config
---
apiVersion: v1
kind: Service
metadata:
  name: mock-openai
  namespace: default
spec:
  selector:
    app: mock-openai
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mock-openai-config
  namespace: default
data:
  mappings.json: |
    {
      "mappings": [
        {
          "request": {
            "method": "POST",
            "url": "/v1/chat/completions"
          },
          "response": {
            "status": 200,
            "headers": {
              "Content-Type": "application/json"
            },
            "jsonBody": {
              "id": "chatcmpl-test-{{randomValue type='ALPHANUMERIC' length=10}}",
              "object": "chat.completion",
              "created": {{currentTimestamp}},
              "model": "gpt-4",
              "choices": [
                {
                  "index": 0,
                  "message": {
                    "role": "assistant",
                    "content": "Hello! This is a mock response from the test OpenAI backend. Current time: {{currentTimestamp}}."
                  },
                  "finish_reason": "stop"
                }
              ],
              "usage": {
                "prompt_tokens": 25,
                "completion_tokens": 15,
                "total_tokens": 40
              }
            },
            "transformers": ["response-template"],
            "fixedDelayMilliseconds": 100
          }
        }
      ]
    }
EOF
```

### Configure AgentGateway Route

```bash
kubectl apply -f- <<'EOF'
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mock-openai-backend
  namespace: default
spec:
  openai:
    connectionString: http://mock-openai.default.svc.cluster.local
    authToken:
      secretRef:
        name: mock-openai-secret
        key: api-key
    model:
      default: gpt-4
      mapping:
        "gpt-4": gpt-4
        "gpt-3.5-turbo": gpt-3.5-turbo
---
apiVersion: v1
kind: Secret
metadata:
  name: mock-openai-secret
  namespace: default
type: Opaque
stringData:
  api-key: "test-api-key"
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mock-openai-route
  namespace: default
spec:
  parentRefs:
  - name: agentgateway
    namespace: enterprise-agentgateway
  hostnames:
  - "*"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /openai
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /v1/chat/completions
    backendRefs:
    - name: mock-openai-backend
      group: agentgateway.dev
      kind: AgentgatewayBackend
      port: 80
EOF
```

### Generate Test Traffic

```bash
# Send test requests to generate observability data
for i in {1..10}; do
  curl -X POST http://localhost:8080/openai \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer test-token" \
    -d '{
      "model": "gpt-4",
      "messages": [
        {"role": "user", "content": "Test message '${i}' for observability demo"}
      ],
      "max_tokens": 50
    }' && echo
  sleep 2
done
```

## Viewing Observability Data

### In Grafana

1. **Dashboard Overview**: Navigate to the AgentGateway dashboard
2. **Request Metrics**: See request rates, response times
3. **Token Usage**: Monitor input/output tokens and costs
4. **Error Analysis**: Check error rates and types

### In Tempo (Distributed Tracing)

1. **Go to Explore** in Grafana
2. **Select Tempo** datasource
3. **Search for traces** by service name: `agentgateway`
4. **Analyze trace spans** showing request flow

### Key Traces to Look For

- **HTTP Request Span**: Gateway ingress
- **LLM Provider Span**: Backend communication
- **Auth Span**: Authentication processing
- **Rate Limit Span**: Rate limiting decisions

### Sample Tempo Query

```
{service.name="agentgateway"}
```

## Advanced Monitoring Configuration

### Custom Metrics

Add custom metrics to track specific business KPIs:

```yaml
rawConfig:
  config:
    metrics:
      enabled: true
      customMetrics:
        - name: "user_requests_total"
          type: "counter"
          help: "Total requests per user"
          labels:
            - user: 'jwt.sub // "anonymous"'
            - model: 'llm.requestModel'
            
        - name: "token_cost_dollars"
          type: "histogram"
          help: "Cost in dollars per request"
          buckets: [0.001, 0.01, 0.1, 1.0, 10.0]
          value: 'llm.totalCost'
```

### Alert Rules

Create Prometheus alert rules for critical conditions:

```yaml
groups:
- name: agentgateway.rules
  rules:
  - alert: HighErrorRate
    expr: rate(agentgateway_requests_total{status=~"5.."}[5m]) > 0.1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High error rate detected"
      
  - alert: HighLatency
    expr: histogram_quantile(0.95, rate(agentgateway_request_duration_seconds_bucket[5m])) > 5
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High latency detected"
      
  - alert: HighCost
    expr: increase(agentgateway_token_cost_dollars_total[1h]) > 50
    for: 0m
    labels:
      severity: warning
    annotations:
      summary: "High hourly cost detected"
```

## Monitoring Best Practices

### Resource Planning

Monitor these key metrics for capacity planning:

1. **Request Rate**: Track requests/second trends
2. **Token Velocity**: Monitor tokens/minute per model
3. **Cost Burn Rate**: Track $/hour consumption
4. **Provider Latency**: Monitor P95/P99 response times

### Performance Optimization

Use observability data to optimize:

1. **Route Configuration**: Based on latency patterns
2. **Caching Strategies**: Based on request patterns
3. **Rate Limiting**: Based on usage distribution
4. **Provider Selection**: Based on cost/performance

### Cost Management

Track and alert on:

1. **Daily/Monthly spend** per provider
2. **Cost per user/team**
3. **Token efficiency** (output/input ratio)
4. **Most expensive models/users**

## Troubleshooting

### Missing Traces

If traces aren't appearing in Tempo:

```bash
# Check Tempo is receiving traces
kubectl logs -l app.kubernetes.io/name=tempo -n monitoring

# Verify AgentGateway trace configuration
kubectl get enterpriseagentgatewayparameters agentgateway-params -n enterprise-agentgateway -o yaml

# Check connectivity
kubectl exec -n enterprise-agentgateway deployment/agentgateway -- nc -zv tempo.monitoring.svc.cluster.local 4317
```

### Missing Metrics

If metrics aren't showing in Prometheus:

```bash
# Check Prometheus targets
curl http://localhost:30090/targets

# Verify AgentGateway metrics endpoint
curl http://localhost:30091/metrics

# Check Prometheus configuration
kubectl get configmap prometheus-server -n monitoring -o yaml
```

### Dashboard Issues

If the dashboard isn't loading:

```bash
# Check Grafana pod logs
kubectl logs -l app.kubernetes.io/name=grafana -n monitoring

# Verify datasource connectivity
kubectl exec -n monitoring deployment/grafana -- nc -zv prometheus-server 80
kubectl exec -n monitoring deployment/grafana -- nc -zv tempo 3100
```

## Cleanup

To remove the monitoring stack:

```bash
helm uninstall grafana -n monitoring
helm uninstall prometheus -n monitoring
helm uninstall tempo -n monitoring
kubectl delete namespace monitoring
```

## Next Steps

With comprehensive observability in place, you're ready to:

1. **Set up development environments** with mock providers
2. **Configure real AI provider integrations**
3. **Implement advanced routing strategies**
4. **Add security and rate limiting policies**

In our next blog post, we'll create a mock OpenAI environment for cost-free development and testing.

## Key Takeaways

- **Complete visibility** into AI Gateway performance and costs
- **Real-time monitoring** with Grafana dashboards
- **Distributed tracing** for request flow analysis
- **Cost tracking** and optimization insights
- **Production-ready** monitoring stack for kind clusters

Your AgentGateway is now fully instrumented and ready for production workloads with comprehensive observability!