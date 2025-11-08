# Helm CD with FluxCD

This repository contains FluxCD configuration for deploying Helm charts to Kubernetes clusters with Istio ingress gateway.

Supports:
- **GKE** (Google Kubernetes Engine) with cloud load balancer
- **On-Prem/Local** Kubernetes clusters with MetalLB
- **Ephemeral GKE Clusters** for automated testing in CI/CD pipelines

## Directory Structure

```
helm-cd/
├── clusters/
│   ├── gke/                      # GKE cluster config
│   │   ├── flux-system/          # Flux bootstrap files
│   │   ├── infrastructure.yaml   # Points to infrastructure configs
│   │   └── environments.yaml     # Points to environment configs
│   ├── local/                    # Local/on-prem cluster config
│   │   ├── flux-system/          # Flux bootstrap files
│   │   ├── infrastructure.yaml
│   │   └── environments.yaml
│   └── ephemeral/                # Ephemeral testing cluster config
│       ├── infrastructure.yaml
│       └── environments.yaml
├── infrastructure/
│   ├── sources/                  # Helm repository sources
│   │   ├── bitnami.yaml
│   │   ├── istio.yaml
│   │   └── metallb.yaml
│   ├── istio/                    # Istio installation (GKE)
│   │   ├── namespace.yaml
│   │   ├── base.yaml
│   │   ├── istiod.yaml
│   │   ├── gateway.yaml          # GKE LoadBalancer config
│   │   └── kustomization.yaml
│   ├── istio-local/              # Istio for on-prem
│   │   ├── gateway.yaml          # MetalLB compatible config
│   │   └── kustomization.yaml
│   ├── metallb/                  # MetalLB for on-prem LoadBalancer
│   │   ├── namespace.yaml
│   │   ├── release.yaml
│   │   ├── ipaddresspool.yaml
│   │   └── kustomization.yaml
│   └── ephemeral/                # Minimal infra for testing
│       └── kustomization.yaml
└── environments/
    ├── dev/
    │   ├── namespace.yaml
    │   ├── app-release.yaml
    │   ├── istio-gateway.yaml
    │   ├── istio-virtualservice.yaml
    │   └── kustomization.yaml
    ├── staging/
    │   └── ...
    ├── prod/
    │   └── ...
    ├── local/                    # Local environment
    │   └── ...
    └── ephemeral/                # Ephemeral testing environment
        ├── namespace.yaml
        ├── app-release.yaml
        ├── istio-gateway.yaml
        ├── istio-virtualservice.yaml
        └── kustomization.yaml
```

## Use Cases

### 1. Production Deployments (GKE)
Deploy and manage production applications on Google Kubernetes Engine with Istio service mesh and GitOps automation.

### 2. Local Development (On-Prem/Local)
Run a complete production-like stack on your local network using MetalLB for LoadBalancer services.

### 3. CI/CD Testing (Ephemeral Clusters)
**NEW**: Automatically create temporary GKE clusters for testing in GitHub Actions, then destroy them when tests complete. Perfect for:
- Integration testing on pull requests
- End-to-end testing before deployment
- Isolated test environments
- Cost-effective testing (clusters only exist during test runs)

See [EPHEMERAL-TESTING.md](./EPHEMERAL-TESTING.md) for detailed instructions.

## Prerequisites

### Common Prerequisites
- kubectl configured to access your cluster
- Flux CLI installed: https://fluxcd.io/flux/installation/
- Git repository (GitHub, GitLab, etc.)

### For GKE
- GKE cluster running

### For On-Prem/Local Kubernetes
- Local Kubernetes cluster running (k3s, k0s, microk8s, kubeadm, etc.)
- Available IP address range on your LAN for LoadBalancer services

### For Ephemeral Testing
- GitHub repository with Actions enabled
- GCP service account with appropriate permissions
- GitHub Personal Access Token
- See [EPHEMERAL-TESTING.md](./EPHEMERAL-TESTING.md)

## Bootstrap FluxCD

### Option 1: Bootstrap on GKE

1. Set your GitHub credentials:
```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

2. Bootstrap Flux on your GKE cluster:
```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=helm-cd \
  --branch=main \
  --path=./clusters/gke \
  --personal
```

### Option 2: Bootstrap on Local/On-Prem Cluster

1. **Configure MetalLB IP Address Pool**

   Before bootstrapping, edit the IP address pool in `infrastructure/metallb/ipaddresspool.yaml` to match your LAN network:

   ```yaml
   spec:
     addresses:
       - 192.168.1.100-192.168.1.110  # Replace with your LAN IP range
   ```

2. **Set your GitHub credentials:**
   ```bash
   export GITHUB_TOKEN=<your-token>
   export GITHUB_USER=<your-username>
   ```

3. **Bootstrap Flux on your local cluster:**
   ```bash
   flux bootstrap github \
     --owner=$GITHUB_USER \
     --repository=helm-cd \
     --branch=main \
     --path=./clusters/local \
     --personal
   ```

Both bootstrap methods will:
- Install Flux components in the `flux-system` namespace
- Create deployment keys in GitHub
- Commit the Flux manifests to the repository
- Start syncing from the repository

## What Gets Deployed

### Infrastructure Layer

#### GKE Cluster
1. **Helm Repositories**: Bitnami and Istio chart repositories
2. **Istio Service Mesh**:
   - istio-base (CRDs)
   - istiod (control plane)
   - istio-ingressgateway (ingress gateway with GCP LoadBalancer)

#### Local/On-Prem Cluster
1. **Helm Repositories**: Bitnami, Istio, and MetalLB chart repositories
2. **MetalLB**: Layer 2 load balancer for bare-metal/on-prem Kubernetes
   - Controller and speaker components
   - IPAddressPool with configurable IP range
   - L2Advertisement for ARP-based IP assignment
3. **Istio Service Mesh**:
   - istio-base (CRDs)
   - istiod (control plane)
   - istio-ingressgateway (single replica, MetalLB LoadBalancer)

### Application Environments
Four environments are configured:
- **dev**: Development environment with basic resources
- **staging**: Pre-production with moderate resources
- **prod**: Production with HA configuration, HTTPS redirect, and CORS
- **local**: Local development with minimal resources

Each environment includes:
- Namespace with Istio injection enabled
- Example Nginx Helm release
- Istio Gateway for ingress
- Istio VirtualService for routing

## Customizing the Example Application

### Adding Your Own Application

1. Replace the example app-release.yaml in each environment:
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: your-app
  namespace: flux-system
spec:
  chart:
    spec:
      chart: your-chart
      sourceRef:
        kind: HelmRepository
        name: your-repo
  targetNamespace: dev
  values:
    # Your values here
```

2. Update the VirtualService to point to your service:
```yaml
route:
  - destination:
      host: your-service.dev.svc.cluster.local
      port:
        number: 8080
```

3. Update the Gateway with your domain:
```yaml
hosts:
  - "dev.yourdomain.com"
```

## TLS Certificate Management

For HTTPS, you need to create TLS certificates. Options:

### Option 1: cert-manager (Recommended)
```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Create ClusterIssuer and Certificate in istio-system namespace
```

### Option 2: Manual Certificate
```bash
# Create secret in istio-system namespace
kubectl create secret tls example-app-tls-cert \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key \
  -n istio-system
```

## Monitoring Flux

```bash
# Check Flux components
flux check

# Watch all Flux resources
flux get all

# Watch Kustomizations
flux get kustomizations --watch

# Watch HelmReleases
flux get helmreleases -A --watch

# Suspend/resume reconciliation
flux suspend kustomization environments
flux resume kustomization environments
```

## Accessing Your Applications

### GKE Cluster

After deployment:

1. Get the Istio ingress gateway LoadBalancer IP:
```bash
kubectl get svc -n istio-system istio-ingressgateway
```

2. Point your DNS records to this IP:
   - dev.example.com
   - staging.example.com
   - example.com / www.example.com

### Local/On-Prem Cluster

After deployment:

1. Get the Istio ingress gateway LoadBalancer IP (assigned by MetalLB):
```bash
kubectl get svc -n istio-system istio-ingressgateway
```

2. Configure local DNS or hosts file:

   **Option A: Local DNS (if you have a DNS server on your LAN)**
   - Add A records pointing to the LoadBalancer IP
   - Example: `local.example.lan` → `192.168.1.100`

   **Option B: /etc/hosts file (for testing)**
   ```bash
   # Add to /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows)
   192.168.1.100 local.example.lan
   ```

3. Access your application:
   ```bash
   curl http://local.example.lan
   ```

## Troubleshooting

### Check Flux logs
```bash
flux logs --all-namespaces --follow
```

### Check HelmRelease status
```bash
kubectl describe helmrelease -n flux-system example-app
```

### Check Istio Gateway
```bash
kubectl get gateway -A
kubectl describe gateway example-app-gateway -n dev
```

### Check VirtualService
```bash
kubectl get virtualservice -A
kubectl describe virtualservice example-app -n dev
```

### MetalLB Troubleshooting (Local/On-Prem)

#### Check MetalLB installation
```bash
kubectl get pods -n metallb-system
kubectl logs -n metallb-system -l app.kubernetes.io/component=controller
```

#### Check IP address pool configuration
```bash
kubectl get ipaddresspool -n metallb-system
kubectl describe ipaddresspool default-pool -n metallb-system
```

#### Check LoadBalancer service status
```bash
kubectl get svc -n istio-system istio-ingressgateway -o wide
```

If the `EXTERNAL-IP` shows `<pending>`:
- Check MetalLB controller logs for errors
- Verify your IP address pool matches your LAN network
- Ensure no IP conflicts with other devices on your network

#### Common MetalLB Issues

**IP Address Conflicts**
- Make sure the IP range in `ipaddresspool.yaml` doesn't overlap with your DHCP range
- Reserve the IP range in your router to prevent conflicts

**Network Configuration**
- MetalLB L2 mode requires Layer 2 connectivity between nodes and clients
- Ensure your firewall allows ARP traffic
- If using VLANs, ensure proper VLAN configuration

## Security Considerations

- Store secrets using SOPS or sealed-secrets
- Enable pod security policies
- Use network policies to restrict traffic
- Regularly update Flux and Istio versions
- Use private Helm repositories for proprietary charts

## Adding More Environments

To add a new environment (e.g., 'qa'):

1. Copy an existing environment:
```bash
cp -r environments/dev environments/qa
```

2. Update namespace and values in all files

3. Add to environments/kustomization.yaml:
```yaml
resources:
  - dev
  - staging
  - qa
  - prod
```

## On-Prem/Local Deployment Considerations

### MetalLB Configuration

MetalLB provides LoadBalancer services for bare-metal Kubernetes clusters. Key configuration points:

1. **IP Address Range**: Edit `infrastructure/metallb/ipaddresspool.yaml` to match your network
   - Use addresses outside your DHCP range
   - Coordinate with network administrators
   - Example ranges: `192.168.1.100-192.168.1.110`

2. **L2 Mode vs BGP Mode**: This setup uses L2 (Layer 2) mode
   - L2 mode: Simple, ARP-based, works on most networks
   - BGP mode: More scalable, requires BGP-capable routers

3. **Load Balancer IP Assignment**:
   - IPs are assigned automatically from the pool
   - Optionally specify a specific IP in the Istio gateway HelmRelease
   - Edit `infrastructure/istio-local/gateway.yaml` and uncomment `loadBalancerIP`

### Recommended Local Kubernetes Distributions

- **k3s**: Lightweight, production-ready (recommended for on-prem)
- **k0s**: Zero-friction Kubernetes
- **MicroK8s**: Snap-based, easy installation
- **kubeadm**: Standard Kubernetes installation tool

### Network Requirements

- Layer 2 network connectivity between cluster nodes and clients
- Static IP addresses or reserved DHCP leases for nodes
- Firewall rules allowing:
  - ARP traffic (for MetalLB L2 mode)
  - HTTP/HTTPS traffic to LoadBalancer IPs
  - Istio control plane communication

### DNS Configuration

For production on-prem deployments:
- Use internal DNS server with wildcard records
- Example: `*.local.example.lan` → MetalLB LoadBalancer IP
- Alternatively, use external-dns with your DNS provider

## Next Steps

- Configure SOPS for secret management
- Set up Prometheus/Grafana for monitoring
- Configure automated image updates with Flux
- Implement canary deployments with Flagger
- Add more HelmReleases for your applications
- For on-prem: Set up backup and disaster recovery procedures