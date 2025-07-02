# Quick Start Guide

## 🚀 5-Minute Deployment

This guide gets you from zero to working gRPC service in 5 minutes.

### Prerequisites ✅
- Kubernetes cluster with AWS Load Balancer Controller
- `kubectl`, `helm`, `grpcurl` installed
- SSL certificate in ACM
- DNS access for your domain

### Step 1: Deploy Traefik (2 minutes)
```bash
# Create namespace
kubectl create namespace traefik

# Add Docker Hub credentials (if needed)
kubectl create secret docker-registry creds \
  --docker-server=docker.io \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD \
  --docker-email=YOUR_EMAIL \
  -n traefik

# Deploy Traefik
helm repo add traefik https://helm.traefik.io/traefik
helm upgrade --install test-traefik traefik/traefik \
  --namespace traefik \
  --values configs/traefik-values.yaml \
  --wait

# Verify
kubectl get pods -n traefik
```

### Step 2: Deploy gRPC Service (1 minute)
```bash
# Deploy gRPC service
kubectl apply -f configs/grpc-deployment.yaml

# Verify
kubectl get pods -n traefik -l app=grpcserver
```

### Step 3: Configure SSL Certificate (30 seconds)
```bash
# Find your certificate ARN
aws acm list-certificates --region us-east-1

# Edit configs/alb-grpc.yaml and update:
# alb.ingress.kubernetes.io/certificate-arn: "YOUR_ACTUAL_CERT_ARN"
```

### Step 4: Deploy ALB (1 minute)
```bash
# Deploy ALB
kubectl apply -f configs/alb-grpc.yaml

# Wait for ALB creation
kubectl get ingress -n traefik grpc-service-alb
# Wait until ADDRESS field shows ALB DNS name
```

### Step 5: Configure DNS (30 seconds)
```bash
# Get ALB DNS name
ALB_DNS=$(kubectl get ingress grpc-service-alb -n traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Create CNAME record:
# grpc-service.example.com -> $ALB_DNS
```

### Step 6: Test (30 seconds)
```bash
# Test gRPC service
grpcurl grpc-service.example.com:443 list

# Test method call
grpcurl -d '{"message": "Hello World"}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse
```

## 🎯 Success Criteria

You should see:
```bash
# Service listing
grpc.reflection.v1alpha.ServerReflection
helloworld.helloworld

# Method response
{
  "message": "Thanks for talking to gRPC server!!! Welcome to hello world. Received message is Hello World",
  "received": true
}
```

## 🔧 Key Configuration Points

### ALB Configuration
```yaml
# Use HTTP2, not GRPC protocol
alb.ingress.kubernetes.io/backend-protocol-version: HTTP2
# Health check on NodePort
alb.ingress.kubernetes.io/healthcheck-port: "32505"
```

### Traefik Configuration
```yaml
# Enable health endpoint
additionalArguments:
  - "--ping=true"
# Expose management port
ports:
  traefik:
    expose: true
```

### gRPC Service Configuration
```yaml
# Use h2c scheme for gRPC
annotations:
  traefik.ingress.kubernetes.io/service.serversscheme: h2c
```

## 🚨 Troubleshooting

### ALB Targets Unhealthy
```bash
# Check NodePort mapping
kubectl get svc test-traefik -n traefik
# Update healthcheck-port in alb-grpc.yaml to match NodePort
```

### gRPC Connection Refused
```bash
# Check ALB status
kubectl describe ingress grpc-service-alb -n traefik
# Verify DNS points to ALB
nslookup grpc-service.example.com
```

### Certificate Issues
```bash
# Verify certificate ARN
aws acm describe-certificate --certificate-arn YOUR_ARN
# Check certificate covers your domain
```

## 📚 Next Steps

1. **Read Full Documentation**: Check `README.md` for comprehensive overview
2. **Advanced Configuration**: See `guides/configuration-guide.md`
3. **Load Testing**: Use examples in `guides/testing-guide.md`
4. **Client Development**: Check `examples/grpc-client-examples.md`
5. **Troubleshooting**: Reference `troubleshooting/common-issues.md`

## 🎉 You're Done!

Your gRPC service is now accessible at `grpc-service.example.com:443` with:
- ✅ SSL termination at ALB
- ✅ HTTP/2 support for gRPC
- ✅ Load balancing and health checks
- ✅ Production-ready configuration

For production deployment, review the security and monitoring sections in the full documentation. 