# Socket Firewall Helm Chart

Kubernetes Helm chart for deploying the Socket.dev Registry Firewall. Blocks vulnerable and malicious packages before they reach your cluster.

## Prerequisites

- Kubernetes 1.21+
- Helm 3.8+
- Socket.dev API token with scopes: `packages`, `entitlements:list`
  - Create at: Socket Dashboard → Settings → API Tokens

## Quick Start

```bash
# Add your Socket API token
export SOCKET_API_TOKEN="your-token-here"

# Install the chart
helm install socket-firewall . \
  --set socket.apiToken=$SOCKET_API_TOKEN \
  --set registries.npm.domains[0]=npm.internal.example.com

# Verify deployment
kubectl get pods -l app.kubernetes.io/name=socket-firewall
```

## Configuration

### Required

| Parameter | Description |
|-----------|-------------|
| `socket.apiToken` | Socket.dev API token |

### Registry Configuration

Enable registries and set custom domains for proxying:

```yaml
registries:
  npm:
    enabled: true
    domains:
      - npm.internal.example.com
  pypi:
    enabled: true
    domains:
      - pypi.internal.example.com
  maven:
    enabled: false
  rubygems:
    enabled: false
  cargo:
    enabled: false
  nuget:
    enabled: true
    domains:
      - nuget.internal.example.com
  go:
    enabled: true
    domains:
      - go.internal.example.com
```

### All Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Docker image | `socketdev/socket-registry-firewall` |
| `image.tag` | Image tag | `latest` |
| `replicaCount` | Number of replicas (ignored if autoscaling enabled) | `1` |
| `podDisruptionBudget.enabled` | Keep pods available during node maintenance | `true` |
| `podDisruptionBudget.minAvailable` | Minimum pods that must stay running | `1` |
| `autoscaling.enabled` | Enable HorizontalPodAutoscaler | `false` |
| `autoscaling.minReplicas` | Minimum replicas | `2` |
| `autoscaling.maxReplicas` | Maximum replicas | `10` |
| `autoscaling.targetCPUUtilizationPercentage` | CPU threshold for scaling | `70` |
| `autoscaling.targetMemoryUtilizationPercentage` | Memory threshold (optional) | `nil` |
| `socket.apiToken` | Socket API token | `""` |
| `socket.existingSecret` | Use existing secret | `""` |
| `socket.failOpen` | Allow downloads if API unavailable | `true` |
| `socket.cacheTtl` | Cache TTL in seconds | `600` |
| `registries.npm.enabled` | Enable npm proxy | `false` |
| `registries.npm.domains` | npm proxy domains | `["npm.company.local"]` |
| `registries.pypi.enabled` | Enable PyPI proxy | `false` |
| `registries.maven.enabled` | Enable Maven proxy | `false` |
| `registries.nuget.enabled` | Enable NuGet proxy | `false` |
| `registries.go.enabled` | Enable Go proxy | `false` |
| `tls.generateSelfSigned` | Generate self-signed certs | `true` |
| `tls.existingSecret` | Use existing TLS secret | `""` |
| `service.type` | Service type | `ClusterIP` |
| `ingress.enabled` | Enable Ingress | `false` |
| `ingress.className` | Ingress class (nginx, alb, traefik) | `""` |
| `ingress.hosts` | Ingress hostnames | `[]` |
| `ingress.tls` | Ingress TLS configuration | `[]` |
| `resources.limits.cpu` | CPU limit | `1` |
| `resources.limits.memory` | Memory limit | `768Mi` |
| `imagePullSecrets` | Image pull secrets for private registries | `[]` |
| `podSecurityContext` | Pod security context | `{}` |
| `securityContext` | Container security context | `{}` |

See [values.yaml](values.yaml) for all options.

### Example Configurations

Pre-built configurations for common deployment scenarios:

```bash
# Corporate network (internal DNS + corp CA)
helm install socket-firewall . -f examples/corporate.yaml \
  --set socket.apiToken=$SOCKET_API_TOKEN

# Remote-first (public domain + MDM-pushed configs)
helm install socket-firewall . -f examples/remote-first.yaml \
  --set socket.apiToken=$SOCKET_API_TOKEN \
  --set ingress.hosts[0].host=sfw.yourcompany.com
```

## Proxy Modes

### Transparent Proxy (Default)

The firewall automatically proxies public registries (`registry.npmjs.org`, `pypi.org`, etc.) without any configuration. Just point DNS or `/etc/hosts` at the firewall.

**Do not add public domains to `registries.*.domains`** - they're already included. Adding them causes duplicate server warnings.

### Custom Domain Mode

Use custom domains when you want a different hostname:

```yaml
registries:
  npm:
    enabled: true
    domains:
      - npm.company.internal  # Your custom domain
```

Then configure your package manager to use `https://npm.company.internal/`.

## Using with Package Managers

### npm / yarn / pnpm

```bash
# Point npm to your firewall proxy
npm config set registry https://npm.internal.example.com/

# If using self-signed certificates
npm config set strict-ssl false
# Or trust the CA certificate
npm config set cafile /path/to/socket-ca.crt
```

### pip (PyPI)

```bash
pip config set global.index-url https://pypi.internal.example.com/simple/
pip config set global.trusted-host pypi.internal.example.com
```

### Maven

Add to `~/.m2/settings.xml`:
```xml
<mirrors>
  <mirror>
    <id>socket-central</id>
    <url>https://maven.internal.example.com/</url>
    <mirrorOf>central</mirrorOf>
  </mirror>
</mirrors>
```

### NuGet (dotnet)

```bash
# Add the firewall as a package source
dotnet nuget add source https://nuget.internal.example.com/v3/index.json -n socket-firewall

# Or via NuGet.Config in your project root
```

`NuGet.Config`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="socket-firewall" value="https://nuget.internal.example.com/v3/index.json" />
  </packageSources>
</configuration>
```

### Go

```bash
# Set GOPROXY to use the firewall
export GOPROXY=https://go.internal.example.com,direct

# For self-signed certificates
export GOINSECURE=go.internal.example.com
# Or trust the CA certificate system-wide
```

Add to shell profile (`~/.bashrc`, `~/.zshrc`):
```bash
export GOPROXY=https://go.internal.example.com,direct
```

## Ingress Configuration

Expose the firewall externally using an Ingress controller.

### nginx Ingress

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  hosts:
    - host: sfw.company.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: sfw-tls
      hosts:
        - sfw.company.com
```

### AWS ALB Ingress

```yaml
ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
  hosts:
    - host: sfw.company.com
      paths:
        - path: /
          pathType: Prefix
```

### Transparent Proxy (Multiple Hosts)

Route multiple registry domains through the firewall:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  hosts:
    - host: registry.npmjs.org
      paths:
        - path: /
          pathType: Prefix
    - host: pypi.org
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: registry-tls
      hosts:
        - registry.npmjs.org
        - pypi.org
```

## TLS Configuration

### Self-Signed (Default)

The chart generates self-signed certificates automatically. Extract the CA cert:

```bash
POD=$(kubectl get pod -l app.kubernetes.io/name=socket-firewall -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- cat /etc/nginx/ssl/ca.crt > socket-ca.crt
```

### Existing Certificate

```yaml
tls:
  generateSelfSigned: false
  existingSecret: my-tls-secret  # must contain tls.crt and tls.key
```

### cert-manager

Create a Certificate resource and reference the secret:

```yaml
tls:
  generateSelfSigned: false
  existingSecret: socket-firewall-tls
```

## Autoscaling

Enable horizontal pod autoscaling to handle variable load:

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80  # optional
```

When enabled, the HorizontalPodAutoscaler manages replica count based on CPU and/or memory utilization. The `replicaCount` value is ignored.

**Requirements:**
- Kubernetes metrics-server must be installed
- Resource requests must be set (they are by default)

**Verify autoscaling:**
```bash
kubectl get hpa socket-firewall
kubectl describe hpa socket-firewall
```

## Using an Existing Secret for API Token

```bash
# Create secret
kubectl create secret generic socket-api-token \
  --from-literal=SOCKET_SECURITY_API_TOKEN=your-token

# Reference in values
helm install socket-firewall . \
  --set socket.existingSecret=socket-api-token
```

**Note:** If you update the API token, restart the deployment to pick up the new value:

```bash
kubectl rollout restart deployment/socket-firewall
```

## Deployment Recommendations

### Corporate Network (On-Prem or VPN)

**Best approach: Internal DNS + Corporate CA**

Zero configuration required on developer laptops.

1. **Deploy the firewall** with `service.type=LoadBalancer` or behind an Ingress
2. **Internal DNS** resolves public registry domains to the firewall IP:
   ```
   registry.npmjs.org  →  10.0.0.50 (firewall IP)
   pypi.org            →  10.0.0.50
   crates.io           →  10.0.0.50
   ```
3. **Use a corporate CA certificate** that's already trusted on managed devices:
   ```yaml
   tls:
     generateSelfSigned: false
     existingSecret: corporate-wildcard-tls
   ```

**Result:** Developers run `npm install` as normal. Traffic routes through the firewall automatically.

**Tradeoff:** Only works when developers are on corporate network or VPN.

### Remote-First Companies

**Best approach: Custom domain + MDM-pushed configs**

For companies without a corporate network or VPN requirement.

1. **Deploy the firewall** with a public domain (e.g., `sfw.company.com`)
2. **Use a real SSL certificate** (Let's Encrypt via cert-manager, or commercial CA):
   ```yaml
   tls:
     generateSelfSigned: false
     existingSecret: sfw-company-com-tls
   ```
3. **Push package manager configs via MDM** (Jamf, Intune, etc.):

   `.npmrc`:
   ```
   registry=https://sfw.company.com/npm/
   ```

   `pip.conf`:
   ```ini
   [global]
   index-url = https://sfw.company.com/pypi/simple/
   ```

**Result:** Works from anywhere (home, coffee shop, office). No VPN required.

**Tradeoff:** Requires pushing 4-6 config files per laptop via endpoint management.

### Comparison

| Approach | Laptop Config | Works Remote | VPN Required |
|----------|---------------|--------------|--------------|
| Internal DNS + Corp CA | None | No | Yes |
| MDM + Custom Domain | Package manager configs | Yes | No |
| MDM + Transparent Proxy | /etc/hosts + cert trust | No | Yes |

### Security Considerations

- **Restrict access** to the firewall if exposed publicly (IP allowlist, VPN, or Zero Trust)
- **Use real certificates** in production to avoid cert trust issues
- **Monitor blocked packages** via the `/socket-stats` endpoint or Socket dashboard

## Uninstall

```bash
helm uninstall socket-firewall
```

## Support

- [Socket.dev Documentation](https://docs.socket.dev)
