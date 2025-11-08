# Local/On-Prem Kubernetes Setup Guide

This guide helps you deploy FluxCD with Istio and MetalLB on your local LAN Kubernetes cluster.

## Quick Start

### 1. Prerequisites

- Local Kubernetes cluster running (k3s, k0s, microk8s, etc.)
- `kubectl` configured to access your cluster
- `flux` CLI installed
- GitHub account and personal access token

### 2. Configure MetalLB IP Range

**IMPORTANT**: Before deploying, configure the IP address range for MetalLB.

Edit `infrastructure/metallb/ipaddresspool.yaml`:

```yaml
spec:
  addresses:
    - 192.168.1.100-192.168.1.110  # Change to match your LAN
```

**Tips for choosing an IP range:**
- Use IPs on the same subnet as your cluster nodes
- Avoid your DHCP range (typically .2 to .99 or similar)
- Reserve 5-10 IPs for LoadBalancer services
- Common ranges:
  - `192.168.1.100-192.168.1.110`
  - `192.168.0.100-192.168.0.110`
  - `10.0.0.100-10.0.0.110`

### 3. Bootstrap Flux

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=helm-cd \
  --branch=main \
  --path=./clusters/local \
  --personal
```

### 4. Verify Deployment

Wait for all components to deploy (5-10 minutes):

```bash
# Watch Flux reconciliation
watch flux get kustomizations

# Check infrastructure components
kubectl get pods -n metallb-system
kubectl get pods -n istio-system

# Check application
kubectl get pods -n local
```

### 5. Get LoadBalancer IP

```bash
kubectl get svc -n istio-system istio-ingressgateway
```

The `EXTERNAL-IP` column shows the IP assigned by MetalLB.

### 6. Configure DNS/Hosts

**Option A: /etc/hosts (quick testing)**
```bash
# Add this line to /etc/hosts
192.168.1.100 local.example.lan
```

**Option B: Local DNS Server (recommended)**
- Add A record: `local.example.lan` → `192.168.1.100`
- Or wildcard: `*.local.example.lan` → `192.168.1.100`

### 7. Test Access

```bash
curl http://local.example.lan
```

You should see the Nginx welcome page.

## Customization

### Change Application Domain

Edit `environments/local/istio-gateway.yaml` and `environments/local/istio-virtualservice.yaml`:

```yaml
hosts:
  - "myapp.local"  # Change to your desired hostname
```

### Deploy Your Own Application

Edit `environments/local/app-release.yaml`:

```yaml
spec:
  chart:
    spec:
      chart: your-chart-name
      sourceRef:
        kind: HelmRepository
        name: your-repo
  values:
    # Your application values
```

### Assign Specific LoadBalancer IP

Edit `infrastructure/istio-local/gateway.yaml`:

```yaml
values:
  service:
    type: LoadBalancer
    loadBalancerIP: 192.168.1.100  # Specific IP from your pool
```

## Troubleshooting

### MetalLB Not Assigning IP

```bash
# Check MetalLB logs
kubectl logs -n metallb-system -l app.kubernetes.io/component=controller

# Verify IP pool
kubectl get ipaddresspool -n metallb-system
kubectl describe ipaddresspool default-pool -n metallb-system
```

Common issues:
- IP range not matching your network subnet
- IP conflicts with other devices
- IP range overlapping with DHCP

### Istio Gateway Not Working

```bash
# Check Istio installation
kubectl get pods -n istio-system

# Check gateway configuration
kubectl get gateway -n local
kubectl describe gateway example-app-gateway -n local

# Check virtualservice
kubectl get virtualservice -n local
kubectl describe virtualservice example-app -n local
```

### Application Not Accessible

1. Verify LoadBalancer IP is assigned
2. Check application pods are running: `kubectl get pods -n local`
3. Test from within cluster: `kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl http://example-app-nginx.local.svc.cluster.local:8080`
4. Check Istio sidecar injection: `kubectl get pods -n local -o jsonpath='{.items[*].spec.containers[*].name}'` (should show `istio-proxy`)

## Network Architecture

```
Internet/LAN
    |
    v
[MetalLB LoadBalancer IP: 192.168.1.100]
    |
    v
[Istio Ingress Gateway]
    |
    v
[Istio VirtualService Routes]
    |
    v
[Application Services in 'local' namespace]
```

## Additional Resources

- MetalLB Documentation: https://metallb.universe.tf/
- Istio Documentation: https://istio.io/
- FluxCD Documentation: https://fluxcd.io/
- k3s Documentation: https://k3s.io/ (recommended for on-prem)
