# Complete Deployment Guide

## Prerequisites

### Required Tools
- `kubectl` configured for your Kubernetes cluster
- `helm` v3.x installed
- `grpcurl` for testing gRPC services
- AWS CLI configured with appropriate permissions

### Required Resources
- Kubernetes cluster with nodes labeled `<node-label>=true`
- AWS Load Balancer Controller installed in cluster
- SSL certificate in AWS Certificate Manager (ACM)
- DNS record pointing to ALB (will be created)

### Required Permissions
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:*",
                "acm:DescribeCertificate",
                "acm:ListCertificates"
            ],
            "Resource": "*"
        }
    ]
}
```

## Phase 1: Deploy Traefik Ingress Controller

### Step 1: Create Namespace and Secrets
```bash
# Create traefik namespace
kubectl create namespace traefik

# Create Docker Hub credentials (if using private images)
kubectl create secret docker-registry creds \
  --docker-server=docker.io \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD \
  --docker-email=YOUR_EMAIL \
  -n traefik
```

### Step 2: Deploy Traefik with Helm
```bash
# Navigate to configs directory
cd grpc-alb-traefik-docs/configs/

# Deploy Traefik
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

helm upgrade --install test-traefik traefik/traefik \
  --namespace traefik \
  --values traefik-values.yaml \
  --wait
```

### Step 3: Verify Traefik Deployment
```bash
# Check pods
kubectl get pods -n traefik -l app.kubernetes.io/name=traefik

# Check service and note NodePort mappings
kubectl get svc -n traefik test-traefik

# Test health endpoint
kubectl port-forward svc/test-traefik 9000:9000 -n traefik &
curl http://localhost:9000/ping
# Expected: OK

# Stop port-forward
pkill -f "kubectl port-forward"
```

## Phase 2: Deploy gRPC Service

### Step 1: Deploy gRPC Application
```bash
# Deploy gRPC service
kubectl apply -f grpc-deployment.yaml

# Verify deployment
kubectl get pods -n traefik -l app=grpcserver
kubectl get svc -n traefik grpcserver
```

### Step 2: Test gRPC Service Internally
```bash
# Port forward to gRPC service
kubectl port-forward svc/grpcserver 9000:9000 -n traefik &

# Test service listing
grpcurl -plaintext localhost:9000 list
# Expected output:
# grpc.reflection.v1alpha.ServerReflection
# helloworld.helloworld

# Test gRPC method call
grpcurl -plaintext -d '{"message": "Hello World"}' localhost:9000 helloworld.helloworld/GetServerResponse
# Expected: JSON response with success message

# Stop port-forward
pkill -f "kubectl port-forward"
```

## Phase 3: Configure SSL Certificate

### Step 1: Find Your SSL Certificate ARN
```bash
# List available certificates
aws acm list-certificates --region us-east-1

# Note the CertificateArn for your domain
```

### Step 2: Update ALB Configuration
```bash
# Edit alb-grpc.yaml and update the certificate ARN
# Replace: arn:aws:acm:us-east-1:YOUR_ACCOUNT:certificate/YOUR_CERT_ID
# With your actual certificate ARN
```

## Phase 4: Deploy ALB for External Access

### Step 1: Deploy ALB Ingress
```bash
# Deploy ALB configuration
kubectl apply -f alb-grpc.yaml

# Check ingress creation
kubectl get ingress -n traefik grpc-service-alb
```

### Step 2: Wait for ALB Provisioning
```bash
# Monitor ALB creation (takes 2-3 minutes)
kubectl describe ingress grpc-service-alb -n traefik

# Wait until ADDRESS field shows ALB DNS name
watch kubectl get ingress -n traefik grpc-service-alb
```

### Step 3: Verify Target Health
```bash
# Get target group ARN from ingress description
kubectl describe ingress grpc-service-alb -n traefik | grep TargetGroup

# Check target health
aws elbv2 describe-target-health --target-group-arn <YOUR_TARGET_GROUP_ARN>

# Wait until TargetHealth shows "healthy"
```

## Phase 5: Configure DNS and Test External Access

### Step 1: Update DNS Record
```bash
# Get ALB DNS name
ALB_DNS=$(kubectl get ingress grpc-service-alb -n traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ALB DNS: $ALB_DNS"

# Create DNS CNAME record:
# grpc-service.example.com -> $ALB_DNS
```

### Step 2: Test External gRPC Access
```bash
# Test DNS resolution
nslookup grpc-service.example.com

# Test gRPC service listing
grpcurl grpc-service.example.com:443 list
# Expected output:
# grpc.reflection.v1alpha.ServerReflection
# helloworld.helloworld

# Test gRPC method call
grpcurl -d '{"message": "Hello from external client"}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse

# Expected: JSON response with success message
```

## Phase 6: Validation and Monitoring

### Step 1: Comprehensive Testing
```bash
# Test multiple concurrent calls
for i in {1..5}; do
  grpcurl -d "{\"message\": \"Test $i\"}" \
    grpc-service.example.com:443 \
    helloworld.helloworld/GetServerResponse &
done
wait
```

### Step 2: Monitor Health and Metrics
```bash
# Check ALB target health
aws elbv2 describe-target-health --target-group-arn <TARGET_GROUP_ARN>

# Access Traefik dashboard
kubectl port-forward svc/test-traefik 9000:9000 -n traefik &
# Visit: http://localhost:9000/dashboard/

# Check Traefik metrics
curl http://localhost:9000/metrics
```

## Deployment Verification Checklist

- [ ] Traefik pods running and healthy
- [ ] Traefik `/ping` endpoint responding
- [ ] gRPC service pods running
- [ ] gRPC service responding to internal calls
- [ ] ALB created and provisioned
- [ ] ALB targets showing as healthy
- [ ] DNS record pointing to ALB
- [ ] External gRPC calls working
- [ ] SSL certificate valid and trusted
- [ ] Multiple concurrent calls working

## Post-Deployment Tasks

### Security Hardening
1. Review and restrict ALB security groups
2. Enable ALB access logging
3. Configure WAF rules if needed
4. Review RBAC permissions

### Monitoring Setup
1. Configure CloudWatch alarms for ALB
2. Set up Traefik metrics collection
3. Configure gRPC service monitoring
4. Set up log aggregation

### Backup and Documentation
1. Export all Kubernetes configurations
2. Document custom configurations
3. Create runbooks for common operations
4. Set up configuration backup

## Rollback Procedures

### Emergency Rollback
```bash
# Remove ALB (stops external traffic)
kubectl delete ingress grpc-service-alb -n traefik

# Remove gRPC service
kubectl delete -f grpc-deployment.yaml

# Remove Traefik (if needed)
helm uninstall test-traefik -n traefik
```

### Partial Rollback
```bash
# Rollback to basic HTTP ALB
kubectl apply -f ../examples/basic-test.yaml

# Scale down gRPC service
kubectl scale deployment grpcserver --replicas=0 -n traefik
``` 