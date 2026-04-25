---
description: "Container and orchestration standards covering Dockerfile best practices, Kubernetes manifests, Helm charts, resource management, and security hardening."
applyTo: "**/Dockerfile*,**/*.dockerfile,**/docker-compose*,**/*.yaml,**/*.yml,**/charts/**,**/k8s/**,**/kubernetes/**,**/helm/**"
---

# Container & Kubernetes Standards

## Core Philosophy

Containers are immutable, ephemeral, and disposable. If you can't destroy and recreate a container without data loss or downtime, your architecture is wrong. Treat container images like compiled binaries — build once, promote through environments unchanged.

## Dockerfile Best Practices

### Multi-Stage Builds (Mandatory)

Every production Dockerfile must use multi-stage builds. The build stage contains SDK/tools; the runtime stage contains only the application and minimal runtime.

```dockerfile
# Build stage — has SDK, tools, source
FROM mcr.microsoft.com/dotnet/sdk:8.0@sha256:<pin> AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

# Runtime stage — minimal, no SDK
FROM mcr.microsoft.com/dotnet/aspnet:8.0@sha256:<pin> AS runtime
WORKDIR /app
COPY --from=build /app .
USER app
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Image Rules

- **Pin base image versions with SHA digest** — `FROM node:20@sha256:abc123`, never `FROM node:latest`
- **Use minimal base images:** distroless (preferred for prod), Alpine (when you need a shell), or CBL-Mariner for Microsoft workloads
- **Never use `latest` tag** in production Dockerfiles or Kubernetes manifests
- **Scan images with Trivy or Snyk** in CI — fail the build on HIGH/CRITICAL CVEs
- **Image size matters:** Smaller images = faster pulls = faster deployments = smaller attack surface

### Layer Optimization

- **Order layers from least to most frequently changing:** OS packages → language runtime → dependencies → application code
- **COPY dependency manifests first**, then restore/install, then copy source — this maximizes cache hits
- **Combine RUN commands** with `&&` to reduce layers: `RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*`
- **Use .dockerignore** — exclude `.git`, `node_modules`, `bin/`, `obj/`, test files, docs, IDE configs

### Security in Dockerfiles

- **Run as non-root:** Create a dedicated user and switch to it. `USER app` or `USER 1000:1000`
- **COPY, not ADD:** `ADD` has implicit tar extraction and URL fetching — use `COPY` for predictable behavior
- **No secrets in build args or ENV:** Secrets in `ARG`/`ENV` are visible in image history. Use BuildKit secrets mount: `RUN --mount=type=secret,id=npmrc`
- **No `curl | bash` in Dockerfiles** — download, verify checksum, then install
- **Set `HEALTHCHECK`** in Dockerfile for standalone containers (not needed when K8s probes are configured)

### Common Mistakes

```dockerfile
# BAD: Installing dev dependencies in production image
RUN npm install

# GOOD: Production-only dependencies
RUN npm ci --omit=dev

# BAD: Running as root
ENTRYPOINT ["node", "server.js"]

# GOOD: Non-root user
USER node
ENTRYPOINT ["node", "server.js"]

# BAD: Unpinned base image
FROM python:3.12

# GOOD: Pinned with SHA
FROM python:3.12@sha256:abc123def456
```

## Docker Compose

- Use for local development only — never for production orchestration
- Pin all image versions in `docker-compose.yml`
- Use `.env` file for configuration, committed `.env.example` as template
- Define `healthcheck` for all services
- Use named volumes for persistent data, bind mounts for source code in dev
- Set `restart: unless-stopped` for development services
- Define `depends_on` with `condition: service_healthy` for startup ordering

## Kubernetes Manifests

### Resource Requests and Limits (Mandatory)

Every container MUST have both requests and limits defined. No exceptions.

```yaml
resources:
  requests:
    cpu: "100m"      # Guaranteed CPU — used for scheduling
    memory: "256Mi"  # Guaranteed memory — used for scheduling
  limits:
    cpu: "500m"      # CPU ceiling — throttled, not killed
    memory: "512Mi"  # Memory ceiling — OOMKilled if exceeded
```

**Guidelines:**
- **Requests = what the app needs under normal load.** Set based on observed P50 usage
- **Limits = ceiling for burst.** Set at 2-3x requests for CPU, 1.5-2x for memory
- **Memory limit must always be set** — a container without memory limit can OOMKill the node
- **CPU limit is debatable** — some teams omit CPU limits to avoid throttling. If you omit CPU limits, you MUST set requests accurately
- **QoS classes:** Guaranteed (requests == limits) for critical services, Burstable (requests < limits) for most workloads

### Health Probes (Mandatory)

Every deployment must have readiness and liveness probes. Startup probe for slow-starting apps.

```yaml
readinessProbe:         # Is the app ready to receive traffic?
  httpGet:
    path: /healthz/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

livenessProbe:          # Is the app alive (not deadlocked)?
  httpGet:
    path: /healthz/live
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3

startupProbe:           # Has the app finished starting?
  httpGet:
    path: /healthz/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 30  # 30 * 5s = 150s max startup time
```

**Probe Design:**
- **Readiness:** Checks that the app can serve traffic (DB connected, dependencies reachable). Failing readiness removes pod from service — it does NOT restart
- **Liveness:** Checks that the app process is not deadlocked. Keep it simple — a liveness probe that checks dependencies will cause cascading restarts. Failing liveness RESTARTS the pod
- **Startup:** Use for apps with long initialization. Disables liveness/readiness until startup succeeds
- **Never make liveness depend on external systems** — if the database is down, restarting your pod won't fix it

### Pod Disruption Budgets (Required for Production)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 1    # OR maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

Every production deployment with > 1 replica must have a PDB. This prevents cluster operations (node drain, upgrades) from taking down all your pods simultaneously.

### Namespace Isolation

- One namespace per team or per application boundary
- Apply `ResourceQuota` to prevent one team from consuming all cluster resources
- Apply `LimitRange` to set default requests/limits for pods that don't specify them
- Use `NetworkPolicy` to restrict cross-namespace traffic — default deny, explicit allow

### Labels and Annotations (Required)

Every resource must have these labels:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-service       # Application name
    app.kubernetes.io/version: "1.2.3"       # Semantic version
    app.kubernetes.io/component: api          # Component type (api, worker, web)
    app.kubernetes.io/part-of: my-platform   # Parent application/system
    app.kubernetes.io/managed-by: helm       # Deployment tool
    team: platform-health                    # Owning team
```

### Security

- **Run as non-root:** Set `runAsNonRoot: true` and `runAsUser: 1000` in securityContext
- **Read-only filesystem:** Set `readOnlyRootFilesystem: true`, mount writable volumes only where needed
- **No privileged containers:** `privileged: false` always. No `SYS_ADMIN` capability
- **Drop all capabilities, add only what's needed:**
  ```yaml
  securityContext:
    capabilities:
      drop: ["ALL"]
      add: ["NET_BIND_SERVICE"]  # Only if binding port < 1024
  ```
- **Pod Security Standards:** Enforce `restricted` profile in production namespaces
- **Seccomp and AppArmor:** Use `RuntimeDefault` seccomp profile at minimum
- **Service accounts:** Create dedicated service accounts per deployment. Never use `default`
- **Automount service account token:** Set `automountServiceAccountToken: false` unless the pod needs K8s API access

### Secrets Management

- **Never put secrets in manifests, values.yaml, or ConfigMaps**
- Use Kubernetes Secrets as the minimum baseline (they're base64 encoded, not encrypted at rest by default)
- **Preferred: External Secrets Operator** pulling from Azure Key Vault, HashiCorp Vault, or AWS Secrets Manager
- Enable encryption at rest for etcd (Kubernetes secrets storage)
- Rotate secrets without pod restarts using mounted volumes (not env vars)

## Helm Charts

### Structure

```
charts/my-app/
├── Chart.yaml          # Chart metadata, version, dependencies
├── values.yaml         # Default configuration — well-documented
├── values-dev.yaml     # Dev overrides
├── values-staging.yaml # Staging overrides
├── values-prod.yaml    # Production overrides
├── templates/
│   ├── _helpers.tpl    # Template helpers — DRY
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── serviceaccount.yaml
└── tests/
    └── test-connection.yaml
```

### Standards

- **values.yaml is the single source of configuration truth.** Every configurable value must be in values.yaml with a sensible default
- **Use template helpers** (`_helpers.tpl`) for repeated logic — labels, names, selectors
- **Chart versioning follows semver:** Breaking changes = major, new features = minor, fixes = patch
- **Lint charts in CI:** `helm lint`, `helm template` to catch errors before deployment
- **Document every value** with comments in values.yaml
- **Use `helm test`** to verify deployments post-install

## Deployment Strategies

| Strategy | Use When | Risk | Rollback Speed |
|---|---|---|---|
| **Rolling Update** | Default for most services | Low — gradual | Fast — automatic |
| **Blue-Green** | Zero-downtime required, database-compatible changes | Medium — full parallel env | Instant — switch traffic |
| **Canary** | High-risk changes, need production validation | Lowest — small blast radius | Fast — route away |
| **Recreate** | Breaking changes, dev/test environments | High — full downtime | Slow — redeploy |

### Rolling Update (Default)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # Max extra pods during update
    maxUnavailable: 0    # Zero downtime — always have full capacity
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2        # Never fewer than 2 for HA
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

- **minReplicas ≥ 2** for production services (HA)
- Scale on CPU and/or custom metrics (request rate, queue depth)
- Set scale-down stabilization to prevent flapping: `behavior.scaleDown.stabilizationWindowSeconds: 300`

## Container Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|---|---|---|
| Fat images with build tools | Slow deploys, large attack surface | Multi-stage builds |
| Running as root | Container escape = node compromise | Non-root user, drop capabilities |
| Latest tag | Non-reproducible, silent breaking changes | Pin version + SHA digest |
| Secrets in env/args | Visible in image history and inspect | External secrets, mounted volumes |
| No resource limits | OOMKill node, noisy neighbor | Always set requests AND limits |
| No health probes | Dead pods keep receiving traffic | Readiness + liveness mandatory |
| One giant container | Can't scale components independently | One process per container |
| Logging to files | Logs lost when pod restarts | Log to stdout/stderr |
| Storing state in container | Data loss on restart/reschedule | External state (DB, object storage) |
