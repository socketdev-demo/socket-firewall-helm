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

# Install with path-based routing (recommended)
helm install socket-firewall . \
  --namespace socket-firewall --create-namespace \
  --set socket.apiToken=$SOCKET_API_TOKEN \
  --set pathRouting.enabled=true \
  --set pathRouting.domain=sfw.company.com \
  --set 'pathRouting.routes[0].path=/npm' \
  --set 'pathRouting.routes[0].upstream=https://registry.npmjs.org' \
  --set 'pathRouting.routes[0].registry=npm' \
  --set 'pathRouting.routes[1].path=/pypi' \
  --set 'pathRouting.routes[1].upstream=https://pypi.org' \
  --set 'pathRouting.routes[1].registry=pypi'

# Verify deployment
kubectl get pods -n socket-firewall -l app.kubernetes.io/name=socket-firewall

# Port-forward for testing
kubectl port-forward svc/socket-firewall 8443:443 -n socket-firewall

# Test health
curl -sk https://localhost:8443/health

# Test npm through the firewall
npm install express --registry https://localhost:8443/npm/ --strict-ssl=false
```

For simpler installs, use a values file instead of `--set` flags. See [examples/](examples/) for ready-made configs.

## Configuration

### Required

| Parameter | Description |
|-----------|-------------|
| `socket.apiToken` | Socket.dev API token |

### Registry Configuration

**Path-based routing (recommended):** A single domain with path prefixes for each registry.

```yaml
pathRouting:
  enabled: true
  domain: sfw.company.com
  routes:
    - path: /npm
      upstream: https://registry.npmjs.org
      registry: npm
    - path: /pypi
      upstream: https://pypi.org
      registry: pypi
    - path: /maven
      upstream: https://repo1.maven.org/maven2
      registry: maven
```

**Domain-based routing (alternative):** Each registry gets its own subdomain.

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
```

### All Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Docker image | `socketdev/socket-registry-firewall` |
| `image.tag` | Image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `replicaCount` | Number of replicas (ignored if autoscaling enabled) | `1` |
| `socket.apiToken` | Socket API token | `""` |
| `socket.existingSecret` | Use existing secret | `""` |
| `socket.failOpen` | Allow downloads if API unavailable | `true` |
| `socket.cacheTtl` | Cache TTL in seconds | `600` |
| **Path-Based Routing** | | |
| `pathRouting.enabled` | Enable path-based routing | `false` |
| `pathRouting.domain` | Domain for path routing | `""` |
| `pathRouting.configMode` | Config mode: upstream, middle, or omit for downstream | `""` |
| `pathRouting.routes` | List of path/upstream/registry route objects | `[]` |
| **Domain-Based Routing** | | |
| `registries.<name>.enabled` | Enable registry (npm, pypi, maven, etc.) | `false` |
| `registries.<name>.domains` | Custom domains for registry | `[]` |
| **Integrations** | | |
| `metadataFiltering.enabled` | Filter blocked packages from metadata | `false` |
| `redis.enabled` | Enable Redis caching for API lookups | `false` |
| `splunk.enabled` | Enable Splunk HEC integration | `false` |
| `webhook.enabled` | Enable webhook event delivery | `false` |
| **Infrastructure** | | |
| `tls.generateSelfSigned` | Generate self-signed certs | `true` |
| `tls.existingSecret` | Use existing TLS secret | `""` |
| `service.type` | Service type | `ClusterIP` |
| `ingress.enabled` | Enable Ingress | `false` |
| `ingress.className` | Ingress class (nginx, alb, traefik) | `""` |
| `autoscaling.enabled` | Enable HorizontalPodAutoscaler | `false` |
| `podDisruptionBudget.enabled` | Keep pods available during node maintenance | `true` |
| `extraContainers` | Sidecar containers (auth proxies, log collectors) | `[]` |
| `resources.limits.cpu` | CPU limit | `1` |
| `resources.limits.memory` | Memory limit | `768Mi` |

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

### Path-Based Routing (Recommended)

A single domain serves all registries via URL path prefixes. Simplest to deploy and manage.

```yaml
pathRouting:
  enabled: true
  domain: sfw.company.com
  routes:
    - path: /npm
      upstream: https://registry.npmjs.org
      registry: npm
    - path: /pypi
      upstream: https://pypi.org
      registry: pypi
    - path: /maven
      upstream: https://repo1.maven.org/maven2
      registry: maven
```

| Registry | Path | Upstream |
|----------|------|----------|
| npm | `/npm/` | `registry.npmjs.org` |
| PyPI | `/pypi/` | `pypi.org` |
| Maven | `/maven/` | `repo1.maven.org/maven2` |
| Cargo | `/cargo/` | `index.crates.io` |
| RubyGems | `/rubygems/` | `rubygems.org` |
| NuGet | `/nuget/` | `api.nuget.org` |
| Go | `/go/` | `proxy.golang.org` |
| Conda | `/conda/` | `conda.anaconda.org` |

### Domain-Based Routing

Each registry gets its own subdomain. Use when `pathRouting.enabled` is `false`.

```yaml
registries:
  npm:
    enabled: true
    domains:
      - npm.company.internal
```

Then configure your package manager to use `https://npm.company.internal/`.

### Transparent Proxy

Point internal DNS for public registry domains directly at the firewall IP. No package manager configuration needed, but requires DNS control and trusted TLS certificates matching registry domains.

**Do not add public domains to `registries.*.domains`** when using this approach.

## Using with Package Managers

Replace `sfw.company.com` with your firewall domain. These examples use path-based routing. For domain-based routing, replace the full URL with your custom domain (e.g., `https://npm.company.internal/`).

### npm / yarn / pnpm

```bash
npm config set registry https://sfw.company.com/npm/

# If using self-signed certificates
npm config set strict-ssl false
# Or trust the CA certificate
npm config set cafile /path/to/socket-ca.crt
```

`.npmrc` (push via MDM):
```
registry=https://sfw.company.com/npm/
```

### pip (PyPI)

```bash
pip config set global.index-url https://sfw.company.com/pypi/simple/
pip config set global.trusted-host sfw.company.com
```

`pip.conf` (push via MDM):
```ini
[global]
index-url = https://sfw.company.com/pypi/simple/
```

### Maven

Add to `~/.m2/settings.xml`:
```xml
<mirrors>
  <mirror>
    <id>socket-central</id>
    <url>https://sfw.company.com/maven/</url>
    <mirrorOf>central</mirrorOf>
  </mirror>
</mirrors>
```

### NuGet (dotnet)

```bash
dotnet nuget add source https://sfw.company.com/nuget/v3/index.json -n socket-firewall
```

`NuGet.Config`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="socket-firewall" value="https://sfw.company.com/nuget/v3/index.json" />
  </packageSources>
</configuration>
```

### Go

```bash
export GOPROXY=https://sfw.company.com/go/,direct

# For self-signed certificates
export GOINSECURE=sfw.company.com
```

### Cargo

```toml
# ~/.cargo/config.toml
[registries.socket]
index = "sparse+https://sfw.company.com/cargo/"
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
