# Configuration Guide

## Overview

This guide explains the configuration details for each component in the gRPC + ALB + Traefik setup. Understanding these configurations is crucial for customization and troubleshooting.

## ALB Configuration (`alb-grpc.yaml`)

### Core Annotations

#### Backend Protocol Configuration
```yaml
alb.ingress.kubernetes.io/backend-protocol-version: HTTP2
alb.ingress.kubernetes.io/backend-protocol: HTTPS
```

**Explanation**:
- `HTTP2`: Enables HTTP/2 support while allowing HTTP health checks
- `HTTPS`: Backend communication uses HTTPS (Traefik's websecure port)
- **Why not GRPC**: ALB expects gRPC health checks, but Traefik only provides HTTP health checks

#### Health Check Configuration
```yaml
alb.ingress.kubernetes.io/healthcheck-path: "/ping"
alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
alb.ingress.kubernetes.io/healthcheck-port: "32505" #check the port from service and change
alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
alb.ingress.kubernetes.io/success-codes: "200"
```

**Explanation**:
- `healthcheck-path`: Traefik's health endpoint
- `healthcheck-port`: NodePort for Traefik management interface (not the service port)
- `healthcheck-protocol`: HTTP (not HTTPS) for health checks
- `success-codes`: Expected HTTP response code

#### SSL/TLS Configuration
```yaml
alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:us-east-1:ACCOUNT:certificate/CERT_ID"
alb.ingress.kubernetes.io/ssl-redirect: "443"
alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
```

**Explanation**:
- `certificate-arn`: ACM certificate for SSL termination
- `ssl-redirect`: Force HTTPS redirect
- `listen-ports`: ALB listens on port 443 for HTTPS

#### Target Configuration
```yaml
alb.ingress.kubernetes.io/target-type: "instance"
alb.ingress.kubernetes.io/target-node-labels: <node-label>=true
```

**Explanation**:
- `target-type`: Direct node targeting (not IP mode)
- `target-node-labels`: Only target nodes with specific labels

### Service Backend Configuration
```yaml
spec:
  rules:
    - host: grpc-service.example.com
      http:
        paths:
        - backend:
            service:
              name: test-traefik
              port:
                name: websecure  # Points to Traefik's HTTPS port
          path: /
          pathType: Prefix
```

## Traefik Configuration (`traefik-values.yaml`)

### Essential Arguments
```yaml
additionalArguments:
  - "--ping=true"
  - "--ping.entrypoint=traefik"
  - "--entrypoints.websecure.http2.maxConcurrentStreams=1000"
  - "--log.level=INFO"
```

**Explanation**:
- `--ping=true`: Enables health check endpoint
- `--ping.entrypoint=traefik`: Health check on management port
- `--http2.maxConcurrentStreams`: Optimizes for gRPC concurrent streams
- `--log.level=INFO`: Appropriate logging level

### Port Configuration
```yaml
ports:
  traefik:
    port: 9000
    expose: true        # Critical for ALB health checks
    exposedPort: 9000
  websecure:
    port: 8443
    expose: true
    exposedPort: 443
    tls:
      enabled: true
    forwardedHeaders:
      insecure: true    # Trust ALB forwarded headers
```

**Explanation**:
- `traefik.expose: true`: Essential for ALB health checks
- `websecure.tls.enabled`: Enables TLS termination at Traefik
- `forwardedHeaders.insecure`: Trusts X-Forwarded headers from ALB

### Service Configuration
```yaml
service:
  type: NodePort
  annotations:
    service.kubernetes.io/topology-aware-hints: "auto"
```

**Explanation**:
- `NodePort`: Required for ALB instance targeting
- `topology-aware-hints`: Optimizes traffic routing

### Node Selection
```yaml
nodeSelector:
  <node-label>: "true"
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app.kubernetes.io/instance
              operator: In
              values:
                - test-traefik
        topologyKey: kubernetes.io/hostname
```

**Explanation**:
- `nodeSelector`: Ensures pods run on specific nodes
- `podAntiAffinity`: Prevents multiple Traefik pods on same node

## gRPC Service Configuration (`grpc-deployment.yaml`)

### Deployment Configuration
```yaml
spec:
  replicas: 1
  template:
    spec:
      nodeSelector:
        kubernetes.io/arch: amd64
        kubernetes.io/os: linux
      containers:
      - name: grpc-demo
        image: pikilisaikiran/grpc:1.0.2
        ports:
        - name: grpc-api
          containerPort: 9000
```

**Explanation**:
- `nodeSelector`: Ensures compatibility with node architecture
- `containerPort: 9000`: gRPC service port

### Service Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    traefik.ingress.kubernetes.io/service.serversscheme: h2c
spec:
  type: ClusterIP
  ports:
    - name: grpc-api
      port: 9000
      targetPort: 9000
      protocol: TCP
```

**Key Configuration**:
- `service.serversscheme: h2c`: Enables HTTP/2 cleartext for gRPC
- `ClusterIP`: Internal service type

### IngressRoute Configuration
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: grpcserver-route
  namespace: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`grpc-service.example.com`)
      kind: Rule
      services:
        - name: grpcserver
          port: 9000
          scheme: h2c  # HTTP/2 cleartext for gRPC
  tls:
    secretName: ""  # Let Traefik handle TLS
```

**Key Configuration**:
- `entryPoints: websecure`: Uses HTTPS entry point
- `scheme: h2c`: HTTP/2 cleartext for gRPC communication
- `tls.secretName: ""`: Traefik handles TLS automatically

## Port Mapping Reference

### Traffic Flow Ports
```
Client:443 → ALB:443 → Node:30137 → Traefik:8443 → gRPC:9000
```

### Health Check Ports
```
ALB Health Check → Node:32505 → Traefik:9000 (/ping)
```

### Port Mapping Table
| Component | Internal Port | NodePort | Purpose |
|-----------|---------------|----------|---------|
| Traefik Management | 9000 | 32505 | Health checks, dashboard |
| Traefik Web | 8000 | 31377 | HTTP traffic |
| Traefik WebSecure | 8443 | 30137 | HTTPS traffic (gRPC) |
| Traefik Metrics | 9100 | 31948 | Prometheus metrics |
| gRPC Service | 9000 | N/A | gRPC application |

## Environment-Specific Configurations

### Development Environment
```yaml
# Reduced resource requirements
resources:
  requests:
    cpu: 50m
    memory: 100Mi
  limits:
    cpu: 100m
    memory: 200Mi

# Debug logging
additionalArguments:
  - "--log.level=DEBUG"
```

### Production Environment
```yaml
# Higher resource allocation
resources:
  requests:
    cpu: 200m
    memory: 300Mi
  limits:
    cpu: 500m
    memory: 500Mi

# Production logging
additionalArguments:
  - "--log.level=INFO"
  - "--accesslog=true"
  - "--metrics.prometheus=true"
```

### High Availability Configuration
```yaml
# Multiple replicas
replicas: 3

# Pod disruption budget
podDisruptionBudget:
  enabled: true
  minAvailable: 2

# Autoscaling
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
```

## Security Configurations

### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: grpc-network-policy
  namespace: traefik
spec:
  podSelector:
    matchLabels:
      app: grpcserver
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: traefik
    ports:
    - protocol: TCP
      port: 9000
```

### RBAC Configuration
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: grpc-service-role
  namespace: traefik
rules:
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch"]
```

## Monitoring and Observability

### Prometheus Metrics
```yaml
# Traefik metrics configuration
metrics:
  prometheus:
    entryPoint: metrics
    addRoutersLabels: true
    buckets: "0.1,0.3,1.2,5.0"
```

### Health Check Endpoints
- **Traefik Health**: `http://node:32505/ping`
- **Traefik Dashboard**: `http://node:32505/dashboard/`
- **Traefik API**: `http://node:32505/api/`
- **Traefik Metrics**: `http://node:31948/metrics`

### Logging Configuration
```yaml
logs:
  general:
    level: INFO
  access:
    enabled: true
    format: json
    fields:
      general:
        defaultmode: keep
      headers:
        defaultmode: drop
```

## Customization Guidelines

### Adding Custom Headers
```yaml
# In IngressRoute
middlewares:
  - name: custom-headers
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: custom-headers
spec:
  headers:
    customRequestHeaders:
      X-Custom-Header: "gRPC-Service"
```

### Rate Limiting
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
spec:
  rateLimit:
    burst: 100
    average: 50
```

### Circuit Breaker
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: circuit-breaker
spec:
  circuitBreaker:
    expression: NetworkErrorRatio() > 0.3
```

## Configuration Validation

### Pre-deployment Checks
```bash
# Validate YAML syntax
kubectl apply --dry-run=client -f alb-grpc.yaml
kubectl apply --dry-run=client -f grpc-deployment.yaml

# Check certificate ARN
aws acm describe-certificate --certificate-arn YOUR_CERT_ARN

# Validate node labels
kubectl get nodes -l <node-label>=true
```

### Post-deployment Validation
```bash
# Check all resources
kubectl get all -n traefik

# Validate IngressRoute
kubectl get ingressroute -n traefik

# Check ALB creation
kubectl describe ingress grpc-service-alb -n traefik
``` 