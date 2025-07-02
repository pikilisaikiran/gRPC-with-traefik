# Testing Guide

## Overview

This guide provides comprehensive testing procedures for validating your gRPC + ALB + Traefik setup. Tests are organized by component and complexity level.

## Prerequisites

### Required Tools
```bash
# Install grpcurl
curl -sSL "https://github.com/fullstorydev/grpcurl/releases/download/v1.8.7/grpcurl_1.8.7_linux_x86_64.tar.gz" | tar -xz -C /usr/local/bin

# Install kubectl
# Install curl, jq for API testing
sudo apt-get install curl jq

# Verify tools
grpcurl --version
kubectl version --client
```

### Test Environment Setup
```bash
# Set environment variables
export GRPC_HOST="grpc-service.example.com"
export GRPC_PORT="443"
export NAMESPACE="traefik"
```

## Level 1: Component Testing

### 1.1 Traefik Health Testing
```bash
# Test Traefik health endpoint directly
kubectl port-forward svc/test-traefik 9000:9000 -n traefik &
curl -i http://localhost:9000/ping

# Expected Response:
# HTTP/1.1 200 OK
# Content-Type: text/plain; charset=utf-8
# Date: ...
# Content-Length: 2
# 
# OK

# Test Traefik dashboard
curl -s http://localhost:9000/api/overview | jq .

# Stop port-forward
pkill -f "kubectl port-forward"
```

### 1.2 gRPC Service Internal Testing
```bash
# Port forward to gRPC service
kubectl port-forward svc/grpcserver 9001:9000 -n traefik &

# Test service reflection
grpcurl -plaintext localhost:9001 list
# Expected: grpc.reflection.v1alpha.ServerReflection, helloworld.helloworld

# Test service description
grpcurl -plaintext localhost:9001 describe helloworld.helloworld

# Test method call
grpcurl -plaintext -d '{"message": "Internal Test"}' \
  localhost:9001 helloworld.helloworld/GetServerResponse

# Expected JSON response with success message

# Stop port-forward
pkill -f "kubectl port-forward"
```

### 1.3 ALB Health Check Testing
```bash
# Get node IP and test health check endpoint
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
HEALTH_PORT=$(kubectl get svc test-traefik -n traefik -o jsonpath='{.spec.ports[?(@.name=="traefik")].nodePort}')

curl -i http://$NODE_IP:$HEALTH_PORT/ping
# Expected: HTTP/1.1 200 OK with "OK" body
```

## Level 2: Integration Testing

### 2.1 ALB to Traefik Integration
```bash
# Get ALB DNS name
ALB_DNS=$(kubectl get ingress grpc-service-alb -n traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test ALB health (should get 404 - means ALB->Traefik working)
curl -H "Host: grpc-service.example.com" http://$ALB_DNS/
# Expected: 404 page not found (from Traefik)

# Test HTTPS redirect
curl -i -H "Host: grpc-service.example.com" http://$ALB_DNS/
# Expected: 301 redirect to HTTPS
```

### 2.2 End-to-End gRPC Testing
```bash
# Test gRPC service listing through ALB
grpcurl $GRPC_HOST:$GRPC_PORT list
# Expected: Service list including helloworld.helloworld

# Test gRPC method call through ALB
grpcurl -d '{"message": "External Test"}' \
  $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse
# Expected: JSON response with success message
```

## Level 3: Load and Performance Testing

### 3.1 Concurrent gRPC Calls
```bash
#!/bin/bash
# concurrent_test.sh

# Function to make gRPC call
make_grpc_call() {
  local id=$1
  echo "Starting call $id"
  result=$(grpcurl -d "{\"message\": \"Load test $id\"}" \
    $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse 2>&1)
  echo "Call $id completed: $result"
}

# Run 10 concurrent calls
for i in {1..10}; do
  make_grpc_call $i &
done

# Wait for all calls to complete
wait
echo "All concurrent calls completed"
```

### 3.2 Sustained Load Testing
```bash
#!/bin/bash
# sustained_load.sh

# Run continuous load for 5 minutes
end_time=$((SECONDS + 300))
call_count=0
success_count=0

while [ $SECONDS -lt $end_time ]; do
  call_count=$((call_count + 1))
  
  if grpcurl -d "{\"message\": \"Load test $call_count\"}" \
     $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse > /dev/null 2>&1; then
    success_count=$((success_count + 1))
  fi
  
  # Brief pause between calls
  sleep 0.1
done

echo "Load test completed:"
echo "Total calls: $call_count"
echo "Successful calls: $success_count"
echo "Success rate: $(( success_count * 100 / call_count ))%"
```

## Level 4: Failure Testing

### 4.1 gRPC Service Failure Simulation
```bash
# Scale down gRPC service
kubectl scale deployment grpcserver --replicas=0 -n traefik

# Test behavior during service unavailability
grpcurl $GRPC_HOST:$GRPC_PORT list
# Expected: Connection error or 503 Service Unavailable

# Scale back up
kubectl scale deployment grpcserver --replicas=1 -n traefik

# Wait for pod to be ready
kubectl wait --for=condition=ready pod -l app=grpcserver -n traefik --timeout=60s

# Test recovery
grpcurl $GRPC_HOST:$GRPC_PORT list
# Expected: Service list returns successfully
```

### 4.2 Traefik Failure Simulation
```bash
# Scale down Traefik
kubectl scale deployment test-traefik --replicas=0 -n traefik

# Test ALB target health
aws elbv2 describe-target-health --target-group-arn $(
  kubectl describe ingress grpc-service-alb -n traefik | 
  grep TargetGroup | awk '{print $2}'
)
# Expected: Targets show as unhealthy

# Scale back up
kubectl scale deployment test-traefik --replicas=1 -n traefik

# Wait for recovery
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=traefik -n traefik --timeout=120s
```

## Level 5: Security Testing

### 5.1 SSL/TLS Validation
```bash
# Test SSL certificate
openssl s_client -connect $GRPC_HOST:443 -servername $GRPC_HOST < /dev/null

# Check certificate chain
curl -vI https://$GRPC_HOST 2>&1 | grep -E "(SSL|TLS|certificate)"

# Test SSL Labs rating (if publicly accessible)
# https://www.ssllabs.com/ssltest/analyze.html?d=grpc-service.example.com
```

### 5.2 Protocol Security Testing
```bash
# Test that HTTP is redirected to HTTPS
curl -I http://$GRPC_HOST
# Expected: 301 redirect to HTTPS

# Test direct gRPC over HTTP (should fail)
grpcurl -plaintext $GRPC_HOST:80 list
# Expected: Connection refused or error

# Test with invalid hostname (should fail)
grpcurl -authority "invalid.example.com" $GRPC_HOST:443 list
# Expected: Certificate validation error
```

## Level 6: Monitoring and Observability Testing

### 6.1 Metrics Collection
```bash
# Test Traefik metrics endpoint
kubectl port-forward svc/test-traefik 9100:9100 -n traefik &

# Fetch Prometheus metrics
curl -s http://localhost:9100/metrics | grep traefik

# Test specific metrics
curl -s http://localhost:9100/metrics | grep -E "(traefik_service_requests_total|traefik_entrypoint_requests_total)"

pkill -f "kubectl port-forward"
```

### 6.2 Health Check Monitoring
```bash
# Monitor ALB target health continuously
watch -n 5 'aws elbv2 describe-target-health --target-group-arn $(
  kubectl describe ingress grpc-service-alb -n traefik | 
  grep TargetGroup | awk "{print \$2}"
) --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]" --output table'
```

## Automated Test Suite

### Complete Test Script
```bash
#!/bin/bash
# complete_test_suite.sh

set -e

echo "=== gRPC + ALB + Traefik Test Suite ==="

# Configuration
GRPC_HOST="grpc-service.example.com"
GRPC_PORT="443"
NAMESPACE="traefik"

# Test 1: Component Health
echo "Test 1: Component Health Checks"
kubectl get pods -n $NAMESPACE
echo "✓ All pods running"

# Test 2: Internal gRPC
echo "Test 2: Internal gRPC Service"
kubectl port-forward svc/grpcserver 9001:9000 -n $NAMESPACE &
PF_PID=$!
sleep 2
grpcurl -plaintext localhost:9001 list > /dev/null
echo "✓ Internal gRPC service responding"
kill $PF_PID

# Test 3: External gRPC
echo "Test 3: External gRPC Access"
grpcurl $GRPC_HOST:$GRPC_PORT list > /dev/null
echo "✓ External gRPC access working"

# Test 4: gRPC Method Call
echo "Test 4: gRPC Method Call"
RESPONSE=$(grpcurl -d '{"message": "Test Suite"}' \
  $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse)
if echo "$RESPONSE" | grep -q "Thanks for talking"; then
  echo "✓ gRPC method call successful"
else
  echo "✗ gRPC method call failed"
  exit 1
fi

# Test 5: Load Test
echo "Test 5: Concurrent Load Test"
for i in {1..5}; do
  grpcurl -d "{\"message\": \"Concurrent $i\"}" \
    $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse > /dev/null &
done
wait
echo "✓ Concurrent calls completed successfully"

# Test 6: ALB Health
echo "Test 6: ALB Target Health"
TARGET_GROUP_ARN=$(kubectl describe ingress grpc-service-alb -n $NAMESPACE | grep TargetGroup | awk '{print $2}')
HEALTH_STATUS=$(aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN --query 'TargetHealthDescriptions[0].TargetHealth.State' --output text)
if [ "$HEALTH_STATUS" = "healthy" ]; then
  echo "✓ ALB targets are healthy"
else
  echo "✗ ALB targets are not healthy: $HEALTH_STATUS"
  exit 1
fi

echo "=== All Tests Passed! ==="
```

## Performance Benchmarking

### Baseline Performance Test
```bash
#!/bin/bash
# benchmark.sh

# Warm-up phase
echo "Warming up..."
for i in {1..10}; do
  grpcurl -d '{"message": "warmup"}' $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse > /dev/null
done

# Benchmark phase
echo "Running benchmark..."
START_TIME=$(date +%s.%N)

for i in {1..100}; do
  grpcurl -d "{\"message\": \"benchmark $i\"}" $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse > /dev/null
done

END_TIME=$(date +%s.%N)
DURATION=$(echo "$END_TIME - $START_TIME" | bc)
RPS=$(echo "100 / $DURATION" | bc -l)

echo "Benchmark Results:"
echo "Total time: ${DURATION}s"
echo "Requests per second: ${RPS}"
```

## Test Data and Scenarios

### Test Message Variations
```bash
# Small message
grpcurl -d '{"message": "small"}' $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse

# Large message
LARGE_MSG=$(python3 -c "print('x' * 1000)")
grpcurl -d "{\"message\": \"$LARGE_MSG\"}" $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse

# Unicode message
grpcurl -d '{"message": "Hello 世界 🌍"}' $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse

# Empty message
grpcurl -d '{"message": ""}' $GRPC_HOST:$GRPC_PORT helloworld.helloworld/GetServerResponse
```

## Troubleshooting Tests

### Diagnostic Commands
```bash
# Check all resource status
kubectl get all -n traefik

# Check ingress details
kubectl describe ingress grpc-service-alb -n traefik

# Check Traefik logs
kubectl logs -l app.kubernetes.io/name=traefik -n traefik --tail=100

# Check gRPC service logs
kubectl logs -l app=grpcserver -n traefik --tail=100

# Check ALB target group health
aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN

# Test DNS resolution
nslookup $GRPC_HOST
dig $GRPC_HOST
```

## Continuous Testing

### Monitoring Script
```bash
#!/bin/bash
# continuous_monitor.sh

while true; do
  timestamp=$(date '+%Y-%m-%d %H:%M:%S')
  
  if grpcurl $GRPC_HOST:$GRPC_PORT list > /dev/null 2>&1; then
    echo "$timestamp - ✓ gRPC service healthy"
  else
    echo "$timestamp - ✗ gRPC service failed"
  fi
  
  sleep 30
done
```

This comprehensive testing guide ensures your gRPC + ALB + Traefik setup is robust, performant, and reliable across various scenarios and failure conditions. 