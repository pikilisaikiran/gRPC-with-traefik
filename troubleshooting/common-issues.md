# Common Issues and Solutions

## Overview

This document covers the most common issues encountered when setting up gRPC with ALB and Traefik, along with their solutions and prevention strategies.

## Issue Categories

- [ALB Health Check Issues](#alb-health-check-issues)
- [gRPC Connection Issues](#grpc-connection-issues)
- [SSL/TLS Certificate Issues](#ssltls-certificate-issues)
- [Traefik Configuration Issues](#traefik-configuration-issues)
- [DNS and Routing Issues](#dns-and-routing-issues)
- [Performance Issues](#performance-issues)

## ALB Health Check Issues

### Issue 1: ALB Targets Showing as Unhealthy

**Symptoms:**
```bash
aws elbv2 describe-target-health --target-group-arn <ARN>
# Output shows: "State": "unhealthy", "Reason": "Target.FailedHealthChecks"
```

**Common Causes:**
1. **Wrong health check port**: Using service port instead of NodePort
2. **Incorrect health check path**: Wrong endpoint or path
3. **Traefik not exposing management port**: Port not exposed in service

**Solutions:**

```bash
# 1. Check NodePort mapping
kubectl get svc test-traefik -n traefik
# Note the NodePort for traefik management port (usually 32xxx)

# 2. Update ALB health check port
# In alb-grpc.yaml, ensure:
alb.ingress.kubernetes.io/healthcheck-port: "32505"  # Use actual NodePort

# 3. Test health check manually
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
curl http://$NODE_IP:32505/ping
# Should return: OK
```

**Prevention:**
- Always verify NodePort mappings after Traefik deployment
- Use consistent health check configuration across environments
- Implement health check monitoring

### Issue 2: Health Check Timeout

**Symptoms:**
```
"Reason": "Target.Timeout"
"Description": "Health check timed out"
```

**Solutions:**
```bash
# 1. Increase health check timeout in ALB configuration
alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "10"

# 2. Check node connectivity
# Ensure nodes are reachable from ALB
aws ec2 describe-security-groups --group-ids <node-security-group>

# 3. Verify Traefik responsiveness
kubectl exec -n traefik <traefik-pod> -- wget -O- http://localhost:9000/ping
```

## gRPC Connection Issues

### Issue 3: gRPC Connection Refused

**Symptoms:**
```bash
grpcurl grpc-service.example.com:443 list
# Error: failed to connect to grpc-service.example.com:443
```

**Diagnostic Steps:**
```bash
# 1. Check ALB status
kubectl get ingress grpc-service-alb -n traefik

# 2. Check DNS resolution
nslookup grpc-service.example.com

# 3. Test ALB connectivity
curl -I https://grpc-service.example.com
```

**Solutions:**
```bash
# 1. Verify ALB is created and has address
kubectl describe ingress grpc-service-alb -n traefik

# 2. Check target health
aws elbv2 describe-target-health --target-group-arn <ARN>

# 3. Verify DNS CNAME record points to ALB
dig grpc-service.example.com CNAME
```

### Issue 4: gRPC Method Not Found

**Symptoms:**
```bash
grpcurl grpc-service.example.com:443 helloworld.helloworld/SayHello
# Error: service "helloworld.helloworld" does not include a method named "SayHello"
```

**Solutions:**
```bash
# 1. List available services and methods
grpcurl grpc-service.example.com:443 list
grpcurl grpc-service.example.com:443 describe helloworld.helloworld

# 2. Check service definition
kubectl logs -l app=grpcserver -n traefik

# 3. Verify correct method name and parameters
grpcurl grpc-service.example.com:443 describe helloworld.Message
```

### Issue 5: gRPC Reflection Not Working

**Symptoms:**
```bash
grpcurl grpc-service.example.com:443 list
# Error: server does not support the reflection API
```

**Solutions:**
```bash
# 1. Verify reflection is enabled in gRPC server
kubectl logs -l app=grpcserver -n traefik | grep -i reflection

# 2. Test with proto file directly
grpcurl -proto helloworld.proto grpc-service.example.com:443 list

# 3. Check if service is using older gRPC version
kubectl describe pod -l app=grpcserver -n traefik
```

## SSL/TLS Certificate Issues

### Issue 6: SSL Certificate Validation Failed

**Symptoms:**
```bash
grpcurl grpc-service.example.com:443 list
# Error: x509: certificate signed by unknown authority
```

**Solutions:**
```bash
# 1. Verify certificate ARN is correct
aws acm describe-certificate --certificate-arn <your-cert-arn>

# 2. Check certificate covers the domain
aws acm describe-certificate --certificate-arn <your-cert-arn> \
  --query 'Certificate.DomainName'

# 3. Test with insecure connection (debug only)
grpcurl -insecure grpc-service.example.com:443 list

# 4. Check certificate chain
openssl s_client -connect grpc-service.example.com:443 -servername grpc-service.example.com
```

### Issue 7: Certificate ARN Not Found

**Symptoms:**
```
InvalidCertificateArn: The certificate ARN is not valid
```

**Solutions:**
```bash
# 1. List available certificates
aws acm list-certificates --region us-east-1

# 2. Verify certificate is in correct region
aws acm list-certificates --region <your-region>

# 3. Check certificate status
aws acm describe-certificate --certificate-arn <arn> --query 'Certificate.Status'
```

## Traefik Configuration Issues

### Issue 8: Traefik Not Routing to gRPC Service

**Symptoms:**
```bash
grpcurl grpc-service.example.com:443 list
# Returns 404 Not Found
```

**Diagnostic Steps:**
```bash
# 1. Check IngressRoute status
kubectl get ingressroute -n traefik

# 2. Check Traefik logs
kubectl logs -l app.kubernetes.io/name=traefik -n traefik --tail=50

# 3. Check service endpoints
kubectl get endpoints grpcserver -n traefik
```

**Solutions:**
```bash
# 1. Verify IngressRoute configuration
kubectl describe ingressroute grpcserver-route -n traefik

# 2. Check service scheme configuration
kubectl get svc grpcserver -n traefik -o yaml | grep -A5 annotations

# 3. Test internal service connectivity
kubectl exec -it <traefik-pod> -n traefik -- wget -O- http://grpcserver:9000
```

### Issue 9: h2c Scheme Not Working

**Symptoms:**
```
Error: HTTP/2 protocol error
```

**Solutions:**
```bash
# 1. Verify h2c annotation on service
kubectl annotate service grpcserver traefik.ingress.kubernetes.io/service.serversscheme=h2c -n traefik

# 2. Check IngressRoute scheme
kubectl get ingressroute grpcserver-route -n traefik -o yaml | grep scheme

# 3. Verify gRPC service supports h2c
kubectl port-forward svc/grpcserver 9000:9000 -n traefik &
grpcurl -plaintext localhost:9000 list
```

## DNS and Routing Issues

### Issue 10: DNS Resolution Fails

**Symptoms:**
```bash
nslookup grpc-service.example.com
# NXDOMAIN error
```

**Solutions:**
```bash
# 1. Check DNS record exists
dig grpc-service.example.com

# 2. Verify CNAME points to ALB
ALB_DNS=$(kubectl get ingress grpc-service-alb -n traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ALB DNS: $ALB_DNS"

# 3. Update DNS record to point to ALB
# Create CNAME: grpc-service.example.com -> $ALB_DNS

# 4. Test with ALB DNS directly
grpcurl $ALB_DNS:443 list -authority grpc-service.example.com
```

### Issue 11: Wrong Host Header

**Symptoms:**
```
HTTP 404 Not Found from Traefik
```

**Solutions:**
```bash
# 1. Check IngressRoute host matching
kubectl get ingressroute grpcserver-route -n traefik -o yaml | grep -A2 match

# 2. Test with correct host header
grpcurl -H "Host: grpc-service.example.com" <alb-dns>:443 list

# 3. Verify ALB host-based routing
kubectl describe ingress grpc-service-alb -n traefik | grep -A10 Rules
```

## Performance Issues

### Issue 12: Slow gRPC Response Times

**Symptoms:**
```
gRPC calls taking longer than expected
```

**Diagnostic Steps:**
```bash
# 1. Test direct service performance
kubectl port-forward svc/grpcserver 9000:9000 -n traefik &
time grpcurl -plaintext localhost:9000 list

# 2. Test through Traefik
kubectl port-forward svc/test-traefik 8443:8443 -n traefik &
time grpcurl -insecure localhost:8443 list -authority grpc-service.example.com

# 3. Test full path
time grpcurl grpc-service.example.com:443 list
```

**Solutions:**
```bash
# 1. Increase HTTP/2 concurrent streams
# In traefik-values.yaml:
additionalArguments:
  - "--entrypoints.websecure.http2.maxConcurrentStreams=2000"

# 2. Optimize resource allocation
# In grpc-deployment.yaml:
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# 3. Enable connection pooling
# In IngressRoute:
services:
  - name: grpcserver
    port: 9000
    scheme: h2c
    strategy: RoundRobin
```

### Issue 13: High Memory Usage

**Symptoms:**
```
Pods getting OOMKilled or high memory consumption
```

**Solutions:**
```bash
# 1. Monitor resource usage
kubectl top pods -n traefik

# 2. Adjust resource limits
kubectl patch deployment grpcserver -n traefik -p '{"spec":{"template":{"spec":{"containers":[{"name":"grpc-demo","resources":{"limits":{"memory":"1Gi"}}}]}}}}'

# 3. Check for memory leaks
kubectl logs -l app=grpcserver -n traefik | grep -i memory
```

## Quick Diagnostic Script

```bash
#!/bin/bash
# diagnostic.sh - Quick health check script

echo "=== gRPC + ALB + Traefik Diagnostic ==="

# 1. Check pods
echo "1. Pod Status:"
kubectl get pods -n traefik

# 2. Check services
echo "2. Service Status:"
kubectl get svc -n traefik

# 3. Check ingress
echo "3. Ingress Status:"
kubectl get ingress -n traefik

# 4. Check ALB target health
echo "4. ALB Target Health:"
TARGET_GROUP_ARN=$(kubectl describe ingress grpc-service-alb -n traefik | grep TargetGroup | awk '{print $2}')
if [ -n "$TARGET_GROUP_ARN" ]; then
  aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' --output table
else
  echo "No target group found"
fi

# 5. Test gRPC connectivity
echo "5. gRPC Connectivity Test:"
if grpcurl grpc-service.example.com:443 list > /dev/null 2>&1; then
  echo "✓ gRPC service accessible"
else
  echo "✗ gRPC service not accessible"
fi

# 6. Check recent logs
echo "6. Recent Traefik Logs:"
kubectl logs -l app.kubernetes.io/name=traefik -n traefik --tail=5

echo "7. Recent gRPC Logs:"
kubectl logs -l app=grpcserver -n traefik --tail=5

echo "=== Diagnostic Complete ==="
```

## Prevention Best Practices

### Configuration Validation
```bash
# Always validate configurations before applying
kubectl apply --dry-run=client -f alb-grpc.yaml
kubectl apply --dry-run=client -f grpc-deployment.yaml

# Use configuration linting
yamllint alb-grpc.yaml
kubeval alb-grpc.yaml
```

### Monitoring Setup
```bash
# Set up alerts for common issues
# - ALB target health
# - gRPC service availability
# - Certificate expiration
# - Resource usage thresholds
```

### Regular Health Checks
```bash
# Implement automated health checks
# - Periodic gRPC calls
# - SSL certificate validation
# - Performance benchmarking
```

This troubleshooting guide should help you quickly identify and resolve the most common issues in your gRPC + ALB + Traefik setup. 