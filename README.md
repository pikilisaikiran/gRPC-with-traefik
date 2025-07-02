# gRPC with AWS ALB and Traefik Ingress Controller

## 📋 Overview

This documentation provides a complete guide for setting up gRPC services behind AWS Application Load Balancer (ALB) using Traefik as the ingress controller on Kubernetes. This setup enables external gRPC access with proper SSL termination, load balancing, and health monitoring.

## 🏗️ Architecture

```
Internet → AWS ALB (HTTPS:443) → Traefik (HTTPS:<nodeport>) → gRPC Service (h2c:<serviceport>)
```

### Traffic Flow
1. **Client** makes gRPC call to `grpc-service.example.com:443`
2. **AWS ALB** receives HTTPS traffic, terminates SSL, forwards as HTTP/2
3. **Traefik** receives HTTP/2 traffic, routes to gRPC service using h2c
4. **gRPC Service** processes request over HTTP/2 cleartext (h2c)

## 🔑 Core Concepts

### 1. **HTTP/2 vs gRPC Protocol Handling**

**Key Insight**: AWS ALB should use `HTTP2` backend protocol, NOT `GRPC` protocol.

- **Why HTTP2?** ALB can perform HTTP health checks while supporting gRPC traffic
- **Why not GRPC?** ALB expects gRPC health checks, but Traefik exposes HTTP health endpoints
- **Result**: ALB handles HTTP/2, Traefik converts to gRPC internally

### 2. **h2c (HTTP/2 Cleartext)**

**Definition**: HTTP/2 over unencrypted connections (without TLS)

- **Purpose**: Allows gRPC communication between Traefik and backend service
- **Configuration**: `scheme: h2c` in Traefik service configuration
- **Benefit**: SSL termination at ALB level, cleartext internally for performance

### 3. **TLS Termination Strategy**

**SSL Termination at ALB Level**:
- ALB handles SSL certificates (ACM integration)
- Internal traffic uses HTTP/2 cleartext (h2c)
- Reduces CPU overhead on Traefik and gRPC services

### 4. **Health Check Configuration**

**Critical Configuration**:
- ALB health checks target Traefik's management port (NodePort)
- Health check endpoint: `/ping` 
- Health check port: `32505` (NodePort for Traefik management)
- Protocol: HTTP (not gRPC)

## 📁 Documentation Structure

```
grpc-with-traefik/
├── README.md                    # This file - main overview
├── configs/                     # Configuration files
│   ├── alb-grpc.yaml           # ALB ingress configuration
│   ├── grpc-deployment.yaml    # gRPC service deployment
│   └── traefik-values.yaml     # Traefik Helm values
├── guides/                      # Step-by-step guides
│   ├── deployment-guide.md     # Complete deployment process
│   ├── testing-guide.md        # Testing procedures
│   └── configuration-guide.md  # Configuration explanations
├── examples/                    # Example configurations
│   ├── basic-test.yaml         # Basic ALB test configuration
│   └── grpc-client-examples.md # gRPC client examples
└── troubleshooting/             # Troubleshooting guides
    ├── common-issues.md        # Common problems and solutions
    └── debugging-guide.md      # Debugging procedures
```

## 🚀 Quick Start

1. **Deploy Traefik**:
   ```bash
   helm upgrade --install test-traefik ./configs/traefik-values.yaml --namespace traefik --create-namespace
   ```

2. **Deploy gRPC Service**:
   ```bash
   kubectl apply -f configs/grpc-deployment.yaml
   ```

3. **Deploy ALB**:
   ```bash
   kubectl apply -f configs/alb-grpc.yaml
   ```

4. **Test gRPC Access**:
   ```bash
   grpcurl grpc-service.example.com:443 list
   ```

## 🔗 Reference Links

### Official Documentation
- [AWS ALB Ingress Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Traefik v2.9 Documentation](https://doc.traefik.io/traefik/v2.9/)
- [gRPC Protocol Documentation](https://grpc.io/docs/)
- [HTTP/2 Specification](https://httpwg.org/specs/rfc7540.html)

### Key Configuration References
- [ALB Annotations Reference](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)
- [Traefik gRPC Configuration](https://doc.traefik.io/traefik/routing/services/#scheme)
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)

### AWS Specific
- [ALB gRPC Support](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#listener-configuration)
- [ACM Certificate Management](https://docs.aws.amazon.com/acm/)
- [ALB Target Groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)

### Traefik Specific
- [Traefik IngressRoute CRD](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)
- [Traefik TLS Configuration](https://doc.traefik.io/traefik/https/tls/)
- [Traefik Health Checks](https://doc.traefik.io/traefik/operations/ping/)

## 📊 Key Metrics & Monitoring

### Health Check Endpoints
- **ALB Health Check**: `http://node:32505/ping`
- **Traefik Dashboard**: `http://localhost:9000/dashboard/`
- **Traefik API**: `http://localhost:9000/api/`

### Important Ports
- **ALB Listener**: 443 (HTTPS)
- **Traefik WebSecure**: 30137 (NodePort)
- **Traefik Management**: 32505 (NodePort)
- **gRPC Service**: 9000 (ClusterIP)

## 🏷️ Tags & Labels

**Resource Tagging Strategy**:
```yaml
alb.ingress.kubernetes.io/tags: "team=test"
```

## 📝 Version Information

- **Traefik Version**: v2.9.5
- **AWS Load Balancer Controller**: Latest
- **Kubernetes**: 1.24+
- **gRPC**: Protocol Buffers v3

## 🤝 Contributing

When updating this documentation:
1. Test all configurations in a non-production environment
2. Update version numbers and compatibility information
3. Add troubleshooting entries for new issues discovered
4. Update reference links if they change

---

**Last Updated**: 01 Jul 2025  
**Maintained By**: Sai Kiran Pikili