---
title: "Setting Up Enterprise AgentGateway on Kind Clusters"
date: 2026-02-09
description: "Complete guide to setting up Enterprise AgentGateway on kind clusters with comprehensive monitoring, security features, and production-ready configurations for development and testing environments."
---


## Introduction

Enterprise AgentGateway by Solo.io is a powerful AI Gateway solution that provides unified access to multiple LLM providers with advanced features like routing, security, observability, and cost management. In this comprehensive guide, we'll walk you through setting up Enterprise AgentGateway on a kind (Kubernetes in Docker) cluster - perfect for development, testing, and learning environments.

## What You'll Learn

- How to set up a kind cluster optimized for AgentGateway
- Installing Kubernetes Gateway API and AgentGateway CRDs
- Deploying the Enterprise AgentGateway controller
- Configuring your first AgentGateway instance
- Validating your installation

## Prerequisites

Before starting this guide, ensure you have:

- **Solo.io Trial License Key**: Get a free trial at [Solo.io](https://www.solo.io/)
- **Docker Desktop** or Docker Engine running
- **kind CLI** installed ([installation guide](https://kind.sigs.k8s.io/docs/user/quick-start/))
- **kubectl CLI** installed and configured
- **helm CLI** installed (version 3.x)

## Kind Cluster Setup

### Create Kind Cluster Configuration

First, let's create a kind cluster with the proper configuration for running AgentGateway:

```bash
cat <<EOF > kind-agentgateway-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: agentgateway
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 8080
    hostPort: 8080
    protocol: TCP
- role: worker
- role: worker
EOF
```

### Create the Cluster

```bash
# Create the kind cluster
kind create cluster --config kind-agentgateway-cluster.yaml

# Verify the cluster is ready
kubectl cluster-info --context kind-agentgateway
kubectl get nodes
```

Expected output:
```bash
NAME                         STATUS   ROLES           AGE   VERSION
agentgateway-control-plane   Ready    control-plane   2m    v1.29.1
agentgateway-worker          Ready    <none>          90s   v1.29.1
agentgateway-worker2         Ready    <none>          90s   v1.29.1
```

## Environment Setup

### Configure Required Variables

Export your Solo.io trial license key and AgentGateway version:

```bash
# Replace with your actual license key from Solo.io
export SOLO_TRIAL_LICENSE_KEY="your-license-key-here"
export ENTERPRISE_AGW_VERSION="2.1.0"

# Verify the variables are set
echo "License Key: $SOLO_TRIAL_LICENSE_KEY"
echo "AgentGateway Version: $ENTERPRISE_AGW_VERSION"
```

## Installing Kubernetes Gateway API

AgentGateway builds on the Kubernetes Gateway API. We'll use the experimental CRDs to enable advanced features:

```bash
# Install Kubernetes Gateway API CRDs (experimental for advanced features)
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/experimental-install.yaml

# Wait for CRDs to be established
kubectl wait --for condition=Established crd/gatewayclasses.gateway.networking.k8s.io
kubectl wait --for condition=Established crd/gateways.gateway.networking.k8s.io
kubectl wait --for condition=Established crd/httproutes.gateway.networking.k8s.io
```

### Verify Gateway API Installation

```bash
kubectl api-resources --api-group=gateway.networking.k8s.io
```

Expected output:
```bash
NAME                 SHORTNAMES   APIVERSION                           NAMESPACED   KIND
backendtlspolicies   btlspolicy   gateway.networking.k8s.io/v1         true         BackendTLSPolicy
gatewayclasses       gc           gateway.networking.k8s.io/v1         false        GatewayClass
gateways             gtw          gateway.networking.k8s.io/v1         true         Gateway
grpcroutes                        gateway.networking.k8s.io/v1         true         GRPCRoute
httproutes                        gateway.networking.k8s.io/v1         true         HTTPRoute
referencegrants      refgrant     gateway.networking.k8s.io/v1beta1    true         ReferenceGrant
tcproutes                         gateway.networking.k8s.io/v1alpha2   true         TCPRoute
tlsroutes                         gateway.networking.k8s.io/v1alpha3   true         TLSRoute
udproutes                         gateway.networking.k8s.io/v1alpha2   true         UDPRoute
```

## Installing Enterprise AgentGateway

### Create Namespace and Install CRDs

```bash
# Create namespace for AgentGateway
kubectl create namespace enterprise-agentgateway

# Install Enterprise AgentGateway CRDs
helm upgrade -i --create-namespace --namespace enterprise-agentgateway \
    --version $ENTERPRISE_AGW_VERSION enterprise-agentgateway-crds \
    oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway-crds

# Wait for CRDs to be established
kubectl wait --for condition=Established crd/agentgatewaybackends.agentgateway.dev
kubectl wait --for condition=Established crd/agentgatewayparameters.agentgateway.dev
kubectl wait --for condition=Established crd/agentgatewaypolicies.agentgateway.dev
```

### Verify AgentGateway CRDs

```bash
kubectl api-resources | awk 'NR==1 || /enterpriseagentgateway\\.solo\\.io|agentgateway\\.dev|ratelimit\\.solo\\.io|extauth\\.solo\\.io/'
```

Expected output:
```bash
NAME                                SHORTNAMES        APIVERSION                                NAMESPACED   KIND
agentgatewaybackends                agbe              agentgateway.dev/v1alpha1                 true         AgentgatewayBackend
agentgatewayparameters              agpar             agentgateway.dev/v1alpha1                 true         AgentgatewayParameters
agentgatewaypolicies                agpol             agentgateway.dev/v1alpha1                 true         AgentgatewayPolicy
enterpriseagentgatewayparameters    eagpar            enterpriseagentgateway.solo.io/v1alpha1   true         EnterpriseAgentgatewayParameters
enterpriseagentgatewaypolicies      eagpol            enterpriseagentgateway.solo.io/v1alpha1   true         EnterpriseAgentgatewayPolicy
authconfigs                         ac                extauth.solo.io/v1                        true         AuthConfig
ratelimitconfigs                    rlc               ratelimit.solo.io/v1alpha1                true         RateLimitConfig
```

## Install Enterprise AgentGateway Controller

```bash
helm upgrade -i -n enterprise-agentgateway enterprise-agentgateway \
    oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway \
    --create-namespace \
    --version $ENTERPRISE_AGW_VERSION \
    --set-string licensing.licenseKey=$SOLO_TRIAL_LICENSE_KEY \
    -f -<<EOF
# Gateway Class parameters reference
gatewayClassParametersRefs:
  enterprise-agentgateway:
    group: enterpriseagentgateway.solo.io
    kind: EnterpriseAgentgatewayParameters
    name: agentgateway-params
    namespace: enterprise-agentgateway
EOF
```

### Verify Controller Installation

```bash
# Check that the controller is running
kubectl get pods -n enterprise-agentgateway -l app.kubernetes.io/name=enterprise-agentgateway

# Wait for the controller to be ready
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=enterprise-agentgateway -n enterprise-agentgateway --timeout=300s
```

Expected output:
```bash
NAME                                       READY   STATUS    RESTARTS   AGE
enterprise-agentgateway-5fc9d95758-n8vvb   1/1     Running   0          87s
```

## Deploy AgentGateway Instance

Now let's create an AgentGateway instance with optimized configuration for kind clusters:

```bash
kubectl apply -f- <<'EOF'
---
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayParameters
metadata:
  name: agentgateway-params
  namespace: enterprise-agentgateway
spec:
  # Enable shared extensions for auth, rate limiting, and caching
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
  
  # Configure logging
  logging:
    level: info
    
  # Configure service for kind cluster (NodePort for local access)
  service:
    spec:
      type: NodePort
      ports:
      - name: http
        port: 8080
        targetPort: 8080
        nodePort: 30080
        protocol: TCP
  
  # Observability configuration
  rawConfig:
    config:
      logging:
        fields:
          add:
            jwt: 'jwt'
            request.body: json(request.body)
            response.body: json(response.body)
            request.body.modelId: json(request.body).modelId
        format: json
      tracing:
        randomSampling: 'true'
        fields:
          add:
            gen_ai.operation.name: '"chat"'
            gen_ai.system: "llm.provider"
            gen_ai.prompt: 'llm.prompt'
            gen_ai.completion: 'llm.completion.map(c, {"role":"assistant", "content": c})'
            gen_ai.request.model: "llm.requestModel"
            gen_ai.response.model: "llm.responseModel"
            gen_ai.usage.completion_tokens: "llm.outputTokens"
            gen_ai.usage.prompt_tokens: "llm.inputTokens"
            gen_ai.request: 'flatten(llm.params)'
            jwt: 'jwt'
            response.body: 'json(response.body)'
            
  # Deployment configuration optimized for kind
  deployment:
    spec:
      replicas: 1  # Single replica for development
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
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway
  namespace: enterprise-agentgateway
spec:
  gatewayClassName: enterprise-agentgateway
  listeners:
    - name: http
      port: 8080
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
EOF
```

## Validation

### Check All Pods Are Running

```bash
kubectl get pods -n enterprise-agentgateway
```

Expected output:
```bash
NAME                                                        READY   STATUS    RESTARTS   AGE
agentgateway-7d4c8c4d4b-lvdsq                               1/1     Running   0          2m
enterprise-agentgateway-5f9c5b95b4-gjblt                    1/1     Running   0          5m
ext-auth-service-enterprise-agentgateway-6fcc5bc989-22wgd   1/1     Running   0          2m
ext-cache-enterprise-agentgateway-6bfcb8c87d-vjzxn          1/1     Running   0          2m
rate-limiter-enterprise-agentgateway-589f66bb88-xz7nm       1/1     Running   0          2m
```

### Check Gateway Status

```bash
kubectl get gateway agentgateway -n enterprise-agentgateway -o yaml
```

Look for the status section showing the gateway is accepted and programmed.

### Test Gateway Accessibility

For kind clusters, we can access the gateway through the NodePort:

```bash
# Get the kind cluster's node port
kubectl get svc -n enterprise-agentgateway --selector=gateway.networking.k8s.io/gateway-name=agentgateway

# Test basic connectivity (should return 404 since no routes are configured yet)
curl -v http://localhost:8080/test
```

Expected response: HTTP 404 (which is correct - no routes configured yet)

## Understanding Your Setup

### What We've Deployed

1. **Kubernetes Gateway API**: Standard APIs for traffic management
2. **Enterprise AgentGateway Controller**: Manages AgentGateway lifecycle
3. **AgentGateway Data Plane**: Routes traffic to AI providers
4. **Shared Extensions**:
   - **ext-auth-service**: Handles authentication (JWT, API keys)
   - **rate-limiter**: Manages rate limiting and quotas  
   - **ext-cache**: Provides caching capabilities

### Network Configuration for Kind

- **Gateway Service**: NodePort on port 30080 for local access
- **Internal Port**: 8080 for cluster-internal communication
- **Host Access**: `http://localhost:8080` from your development machine

## Environment Setup Script

Create a helper script for managing your AgentGateway environment:

```bash
cat <<'EOF' > agentgateway-env.sh
#!/bin/bash

# AgentGateway Environment Setup Script
export SOLO_TRIAL_LICENSE_KEY="${SOLO_TRIAL_LICENSE_KEY:-your-license-key-here}"
export ENTERPRISE_AGW_VERSION="2.1.0"

# Helper functions
agw_status() {
    echo "=== AgentGateway Status ==="
    kubectl get pods -n enterprise-agentgateway
    echo ""
    kubectl get gateway agentgateway -n enterprise-agentgateway
}

agw_logs() {
    kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail=50 -f
}

agw_reset() {
    echo "Resetting AgentGateway..."
    kubectl delete namespace enterprise-agentgateway
    # Re-run installation commands
}

echo "AgentGateway environment loaded!"
echo "Available commands: agw_status, agw_logs, agw_reset"
EOF

# Make it executable and source it
chmod +x agentgateway-env.sh
source agentgateway-env.sh
```

## Troubleshooting

### Common Issues

**1. License Key Issues**
```bash
# Check controller logs for license errors
kubectl logs deploy/enterprise-agentgateway -n enterprise-agentgateway
```

**2. CRD Installation Problems**
```bash
# Reinstall CRDs if needed
kubectl delete crd -l app.kubernetes.io/name=enterprise-agentgateway-crds
# Then re-run the helm install command
```

**3. Pod Startup Issues**
```bash
# Check pod events
kubectl describe pod -l app.kubernetes.io/name=agentgateway -n enterprise-agentgateway

# Check resource constraints
kubectl top pods -n enterprise-agentgateway
```

**4. Gateway Not Ready**
```bash
# Check gateway status
kubectl describe gateway agentgateway -n enterprise-agentgateway

# Verify GatewayClass
kubectl get gatewayclass enterprise-agentgateway -o yaml
```

## Cleanup

When you're done testing, clean up your environment:

```bash
# Delete the AgentGateway installation
helm uninstall enterprise-agentgateway -n enterprise-agentgateway
helm uninstall enterprise-agentgateway-crds -n enterprise-agentgateway

# Delete the namespace
kubectl delete namespace enterprise-agentgateway

# Delete the kind cluster
kind delete cluster --name agentgateway
```

## Next Steps

Now that you have AgentGateway running on your kind cluster, you're ready to:

1. **Set up monitoring and observability** - Install Grafana, Prometheus, and Tempo for comprehensive observability
2. **Configure your first AI route** - Connect to OpenAI, Anthropic, or other providers
3. **Explore advanced features** - Security, rate limiting, and traffic management

In our next blog post, we'll set up a complete observability stack to monitor your AgentGateway's performance, costs, and usage patterns.

## Key Takeaways

- Enterprise AgentGateway provides a unified interface for multiple AI providers
- Kind clusters are perfect for development and testing AgentGateway
- The setup includes controller, data plane, and essential shared extensions
- Proper observability configuration enables comprehensive monitoring
- NodePort service configuration allows easy local access in kind environments

With your AgentGateway foundation in place, you're ready to build sophisticated AI routing and management capabilities!