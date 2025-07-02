# Debugging Guide

## Overview

This guide provides systematic debugging procedures for diagnosing issues in your gRPC + ALB + Traefik setup.

## Debugging Methodology

### 1. **Layer-by-Layer Approach**
Debug from the inside out:
1. gRPC Service → Traefik → ALB → Internet
2. Isolate each component
3. Test each interface independently

### 2. **Information Gathering**
Always collect these before debugging:
- Pod status and logs
- Service configurations
- Network connectivity
- Resource utilization

## Systematic Debugging Procedures

### Phase 1: Environment Assessment

#### Check Cluster Status
```bash
#!/bin/bash
# cluster_status.sh

echo "=== Cluster Status ==="
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces | grep -E "(traefik|grpc)"
```

#### Check Resource Status
```bash
#!/bin/bash
# resource_status.sh

echo "=== Resource Status ==="
echo "Pods:"
kubectl get pods -n traefik -o wide

echo "Services:"
kubectl get svc -n traefik

echo "Ingress:"
kubectl get ingress -n traefik

echo "IngressRoute:"
kubectl get ingressroute -n traefik
```

### Phase 2: Component-Level Debugging

#### Debug gRPC Service
```bash
#!/bin/bash
# debug_grpc_service.sh

echo "=== gRPC Service Debug ==="

# Check pod status
echo "1. Pod Status:"
kubectl describe pod -l app=grpcserver -n traefik

# Check service endpoints
echo "2. Service Endpoints:"
kubectl get endpoints grpcserver -n traefik

# Test internal connectivity
echo "3. Internal Connectivity Test:"
kubectl run debug-pod --image=busybox --rm -it --restart=Never -- sh -c "
  echo 'Testing gRPC service connectivity...'
  nc -zv grpcserver.traefik.svc.cluster.local 9000
"

# Port forward and test
echo "4. Direct Service Test:"
kubectl port-forward svc/grpcserver 9000:9000 -n traefik &
PF_PID=$!
sleep 2

if command -v grpcurl >/dev/null; then
  grpcurl -plaintext localhost:9000 list
else
  curl -v http://localhost:9000/
fi

kill $PF_PID 2>/dev/null
```

#### Debug Traefik
```bash
#!/bin/bash
# debug_traefik.sh

echo "=== Traefik Debug ==="

# Check Traefik pod status
echo "1. Traefik Pod Status:"
kubectl describe pod -l app.kubernetes.io/name=traefik -n traefik

# Check Traefik configuration
echo "2. Traefik Configuration:"
kubectl logs -l app.kubernetes.io/name=traefik -n traefik --tail=20

# Test Traefik health
echo "3. Traefik Health Check:"
kubectl port-forward svc/test-traefik 9000:9000 -n traefik &
PF_PID=$!
sleep 2

curl -v http://localhost:9000/ping
curl -s http://localhost:9000/api/overview | jq .

kill $PF_PID 2>/dev/null

# Check Traefik routes
echo "4. Traefik Routes:"
kubectl port-forward svc/test-traefik 9000:9000 -n traefik &
PF_PID=$!
sleep 2

curl -s http://localhost:9000/api/http/routers | jq .
curl -s http://localhost:9000/api/http/services | jq .

kill $PF_PID 2>/dev/null
```

#### Debug ALB
```bash
#!/bin/bash
# debug_alb.sh

echo "=== ALB Debug ==="

# Get ALB information
echo "1. ALB Configuration:"
kubectl describe ingress grpc-service-alb -n traefik

# Check target group health
echo "2. Target Group Health:"
TARGET_GROUP_ARN=$(kubectl describe ingress grpc-service-alb -n traefik | grep TargetGroup | awk '{print $2}')

if [ -n "$TARGET_GROUP_ARN" ]; then
  aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
  aws elbv2 describe-target-group --target-group-arns $TARGET_GROUP_ARN
else
  echo "Target group ARN not found"
fi

# Test ALB connectivity
echo "3. ALB Connectivity:"
ALB_DNS=$(kubectl get ingress grpc-service-alb -n traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

if [ -n "$ALB_DNS" ]; then
  echo "ALB DNS: $ALB_DNS"
  curl -I -H "Host: grpc-service.example.com" http://$ALB_DNS/
  curl -I https://$ALB_DNS/
else
  echo "ALB DNS not found"
fi
```

### Phase 3: Network Debugging

#### Test Network Connectivity
```bash
#!/bin/bash
# network_debug.sh

echo "=== Network Debug ==="

# Check DNS resolution
echo "1. DNS Resolution:"
nslookup grpc-service.example.com
dig grpc-service.example.com

# Check certificate
echo "2. SSL Certificate:"
openssl s_client -connect grpc-service.example.com:443 -servername grpc-service.example.com < /dev/null 2>/dev/null | openssl x509 -noout -dates -subject

# Test different protocols
echo "3. Protocol Tests:"
# HTTP
curl -I http://grpc-service.example.com/

# HTTPS
curl -I https://grpc-service.example.com/

# gRPC
if command -v grpcurl >/dev/null; then
  grpcurl grpc-service.example.com:443 list
fi
```

#### Port and Service Mapping
```bash
#!/bin/bash
# port_mapping.sh

echo "=== Port Mapping Debug ==="

# Get NodePort mappings
echo "1. NodePort Mappings:"
kubectl get svc test-traefik -n traefik -o yaml | grep -A10 -B5 nodePort

# Test NodePort access
echo "2. NodePort Access Test:"
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
HEALTH_PORT=$(kubectl get svc test-traefik -n traefik -o jsonpath='{.spec.ports[?(@.name=="traefik")].nodePort}')
WEB_PORT=$(kubectl get svc test-traefik -n traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')

echo "Node IP: $NODE_IP"
echo "Health Port: $HEALTH_PORT"
echo "WebSecure Port: $WEB_PORT"

if [ -n "$NODE_IP" ] && [ -n "$HEALTH_PORT" ]; then
  curl -v http://$NODE_IP:$HEALTH_PORT/ping
fi
```

### Phase 4: Advanced Debugging

#### Traffic Flow Analysis
```bash
#!/bin/bash
# traffic_flow.sh

echo "=== Traffic Flow Analysis ==="

# Enable Traefik debug logging
echo "1. Enabling Debug Logging:"
kubectl patch deployment test-traefik -n traefik -p '{"spec":{"template":{"spec":{"containers":[{"name":"traefik","args":["--log.level=DEBUG"]}]}}}}'

# Wait for rollout
kubectl rollout status deployment test-traefik -n traefik

# Make test call and check logs
echo "2. Making Test Call:"
grpcurl -d '{"message": "Debug test"}' grpc-service.example.com:443 helloworld.helloworld/GetServerResponse &

# Check logs immediately
sleep 2
echo "3. Recent Traefik Logs:"
kubectl logs -l app.kubernetes.io/name=traefik -n traefik --tail=50

# Restore normal logging
echo "4. Restoring Normal Logging:"
kubectl patch deployment test-traefik -n traefik -p '{"spec":{"template":{"spec":{"containers":[{"name":"traefik","args":["--log.level=INFO"]}]}}}}'
```

#### Performance Analysis
```bash
#!/bin/bash
# performance_debug.sh

echo "=== Performance Debug ==="

# Check resource usage
echo "1. Resource Usage:"
kubectl top pods -n traefik

# Check limits and requests
echo "2. Resource Configuration:"
kubectl describe pod -l app=grpcserver -n traefik | grep -A5 -B5 "Limits\|Requests"
kubectl describe pod -l app.kubernetes.io/name=traefik -n traefik | grep -A5 -B5 "Limits\|Requests"

# Performance test
echo "3. Performance Test:"
time grpcurl grpc-service.example.com:443 list

# Multiple concurrent calls
echo "4. Concurrent Call Test:"
for i in {1..5}; do
  (time grpcurl -d "{\"message\": \"Perf test $i\"}" grpc-service.example.com:443 helloworld.helloworld/GetServerResponse) &
done
wait
```

## Debugging Tools and Commands

### Essential Commands Reference
```bash
# Quick health check
kubectl get pods -n traefik && \
kubectl get svc -n traefik && \
kubectl get ingress -n traefik

# Get all logs
kubectl logs -l app.kubernetes.io/name=traefik -n traefik --tail=100
kubectl logs -l app=grpcserver -n traefik --tail=100

# Test connectivity
grpcurl grpc-service.example.com:443 list
curl -I https://grpc-service.example.com/

# Check ALB health
aws elbv2 describe-target-health --target-group-arn $(kubectl describe ingress grpc-service-alb -n traefik | grep TargetGroup | awk '{print $2}')
```

### Debugging Environment Variables
```bash
# Set up debugging environment
export NAMESPACE="traefik"
export GRPC_HOST="grpc-service.example.com"
export GRPC_PORT="443"

# Quick aliases
alias kgp="kubectl get pods -n $NAMESPACE"
alias kgs="kubectl get svc -n $NAMESPACE"
alias kgi="kubectl get ingress -n $NAMESPACE"
alias klogs="kubectl logs -n $NAMESPACE"
```

## Common Debugging Scenarios

### Scenario 1: Complete Service Failure
```bash
# Systematic check
1. kubectl get pods -n traefik
2. kubectl get svc -n traefik
3. kubectl get ingress -n traefik
4. kubectl describe ingress grpc-service-alb -n traefik
5. Check ALB target health
6. Check DNS resolution
```

### Scenario 2: Intermittent Failures
```bash
# Continuous monitoring
watch -n 5 'kubectl get pods -n traefik'
watch -n 10 'grpcurl grpc-service.example.com:443 list'

# Log monitoring
kubectl logs -f -l app.kubernetes.io/name=traefik -n traefik
kubectl logs -f -l app=grpcserver -n traefik
```

### Scenario 3: Performance Issues
```bash
# Resource monitoring
kubectl top pods -n traefik
kubectl describe pods -n traefik

# Network latency test
time grpcurl grpc-service.example.com:443 list
time curl -I https://grpc-service.example.com/
```

## Debugging Checklist

### Pre-Debug Information Gathering
- [ ] Collect all pod statuses
- [ ] Gather recent logs from all components
- [ ] Check resource utilization
- [ ] Verify network connectivity
- [ ] Test each component individually

### Systematic Debug Process
- [ ] Test gRPC service directly (port-forward)
- [ ] Test through Traefik (internal)
- [ ] Test through ALB (external)
- [ ] Verify SSL/TLS configuration
- [ ] Check DNS resolution
- [ ] Validate security groups and network policies

### Post-Debug Actions
- [ ] Document findings
- [ ] Implement monitoring for detected issues
- [ ] Update configurations if needed
- [ ] Test fix thoroughly
- [ ] Update troubleshooting documentation

This debugging guide provides a systematic approach to identifying and resolving issues in your gRPC + ALB + Traefik setup. 