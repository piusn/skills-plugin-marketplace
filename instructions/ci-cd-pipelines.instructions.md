---
description: "CI/CD pipeline design standards covering Azure Pipelines, GitHub Actions, deployment strategies, environment promotion, and pipeline security."
applyTo: "**/.pipelines/**,**/.github/workflows/**,**/azure-pipelines*.yml,**/*.pipeline.yml"
---

# CI/CD Pipeline Standards

## Core Philosophy

Pipelines are code. They must be version-controlled, peer-reviewed, tested, and treated with the same rigor as application code. A pipeline that only the original author understands is a liability. If you can't explain every stage and why it exists, simplify it.

**Principles:**
- **Repeatability:** Same commit → same artifact → same deployment result, every time
- **Speed:** Fast feedback loops. CI should complete in < 10 minutes for most repos. If it takes longer, optimize
- **Safety:** Every deployment must be reversible. No deployment without a rollback plan
- **Visibility:** Every team member should understand the pipeline and be able to trigger/rollback deployments

## CI Pipeline Stages

The CI pipeline runs on every push and pull request. It produces a versioned, immutable artifact.

```
┌──────────┐   ┌─────────┐   ┌───────┐   ┌──────┐   ┌──────┐   ┌──────────────┐   ┌──────────────────┐
│ Checkout │──▶│ Restore │──▶│ Build │──▶│ Test │──▶│ Lint │──▶│ Security Scan│──▶│ Publish Artifact │
└──────────┘   └─────────┘   └───────┘   └──────┘   └──────┘   └──────────────┘   └──────────────────┘
```

### Stage Details

1. **Checkout:** Clean checkout. For PRs, checkout the merge commit, not just the branch
2. **Restore:** Restore cached dependencies. Cache key = lockfile hash. If cache miss, install from scratch
3. **Build:** Compile/transpile. Build output is the artifact. Build must be deterministic
4. **Test:** Unit tests + integration tests. Fail fast — run fast unit tests first. Code coverage gating (≥ 80% for new code)
5. **Lint:** Static analysis, formatting checks, style enforcement. This is non-negotiable — fix lint issues, don't disable rules
6. **Security Scan:** Dependency vulnerability scanning (npm audit, dotnet list package --vulnerable, Snyk/Dependabot), SAST for code patterns, secret scanning
7. **Publish Artifact:** Tag with build number + commit SHA. Push to artifact registry (container registry, NuGet feed, npm registry). Artifacts are immutable — never overwrite a published version

### CI Rules

- **CI runs on every PR** — no exceptions, no bypassing for "small changes"
- **CI must pass before merge** — branch protection rules enforce this
- **Tests are not optional** — a PR without tests for new behavior is incomplete
- **Flaky tests are bugs** — quarantine and fix within 1 sprint, or delete them
- **Build once, deploy everywhere** — the same artifact flows through all environments. Environment-specific config is injected at deployment time, not build time

## CD Pipeline Stages

The CD pipeline promotes a CI-produced artifact through environments.

```
┌────────────┐   ┌───────────────────┐   ┌──────────────────┐   ┌─────────────┐   ┌───────────────┐   ┌────────────────────┐   ┌──────────────┐
│ Deploy Dev │──▶│ Integration Tests │──▶│ Deploy Staging   │──▶│ Smoke Tests │──▶│ Approval Gate │──▶│ Deploy Production  │──▶│ Health Check │
└────────────┘   └───────────────────┘   └──────────────────┘   └─────────────┘   └───────────────┘   └────────────────────┘   └──────────────┘
```

### Stage Details

1. **Deploy Dev:** Automatic on merge to main. Deploys latest artifact to dev environment
2. **Integration Tests:** Run API/E2E tests against deployed dev environment. Test real service interactions, not mocks
3. **Deploy Staging:** Automatic if integration tests pass. Staging must mirror production (same config, same scale, production-like data)
4. **Smoke Tests:** Critical path validation — login, core workflows, health endpoints. These are the "if this fails, everything is broken" tests
5. **Approval Gate:** Manual approval for production. Approver must be different from the deployer. Include deployment summary — what changed, what's the risk, what's the rollback plan
6. **Deploy Production:** Use the selected deployment strategy (rolling, blue-green, canary). Monitor during and after deployment
7. **Health Check:** Automated post-deployment validation. If health check fails, trigger automatic rollback

## Environment Promotion

```
Dev → Staging → Production
```

**Rules:**
- **Never skip staging.** "It works in dev" is not a deployment strategy
- **Same artifact, different config.** The binary/image deployed to production is byte-for-byte identical to what was tested in staging
- **Environment parity:** Staging must match production in architecture, configuration shape, and data schema. Scale can differ
- **Config injection:** Use environment variables, Azure App Configuration, Key Vault, or mounted config files. Never bake environment-specific values into the artifact
- **Database migrations run before deployment** — forward-only, backward-compatible

### Environment-Specific Rules

| Environment | Deployment | Testing | Approval | Data |
|---|---|---|---|---|
| **Dev** | Automatic on merge | Integration tests | None | Synthetic/seed data |
| **Staging** | Automatic on dev success | Smoke + perf tests | None | Production-like anonymized data |
| **Production** | After staging success | Health checks | Manual approval required | Real data |

## Deployment Strategies

### Rolling Update (Default)

- Gradually replace old pods with new ones
- **Use when:** Standard deployments, stateless services
- **Config:** `maxSurge: 25%, maxUnavailable: 0` for zero-downtime
- **Rollback:** `kubectl rollout undo` or redeploy previous version

### Blue-Green

- Run old (blue) and new (green) in parallel, switch traffic atomically
- **Use when:** Zero-downtime required, need instant rollback capability
- **Requires:** 2x infrastructure during deployment
- **Rollback:** Route traffic back to blue — instant, no redeployment

### Canary

- Route a small percentage of traffic (5-10%) to the new version, monitor, then gradually increase
- **Use when:** High-risk changes, need production validation with limited blast radius
- **Monitor:** Error rates, latency, business metrics during canary window
- **Rollback:** Route all traffic back to stable version
- **Tools:** Flagger, Argo Rollouts, Azure Traffic Manager weighted routing

### Feature Flags

- Deploy code to production behind a flag, enable for specific users/percentages
- **Use when:** Gradual rollout independent of deployment, A/B testing, kill switch for risky features
- **Tools:** LaunchDarkly, Azure App Configuration feature flags, custom flag service
- **Rules:** Flags are temporary — every flag has an expiration date. Clean up after full rollout

## Pipeline Security

### Secrets Management

- **No secrets in YAML files** — not in pipeline definitions, not in checked-in config
- **Azure Pipelines:** Use Variable Groups linked to Key Vault. Mark variables as secret
- **GitHub Actions:** Use repository/organization/environment secrets. Use OIDC for cloud authentication instead of stored credentials
- **Rotate secrets regularly** — automate rotation where possible
- **Audit secret access** — log who/what accesses secrets and when

### Service Connections and Permissions

- **Least privilege:** Pipeline service principals get only the permissions they need, scoped to the specific resources they deploy to
- **Separate service connections** per environment — dev pipeline cannot deploy to production
- **No shared credentials** between teams — each team/service owns its own service connection
- **Review service connection permissions** quarterly

### Supply Chain Security

- **Lock dependency versions:** Use lockfiles (package-lock.json, packages.lock.json, poetry.lock)
- **Scan dependencies:** npm audit, Snyk, Dependabot, GitHub Advanced Security
- **Pin actions/tasks by SHA:** `uses: actions/checkout@abc123` not `uses: actions/checkout@v4`
- **Sign artifacts:** Sign container images and verify signatures before deployment
- **SBOM generation:** Generate Software Bill of Materials for production artifacts

## Azure Pipelines Specifics

### OneBranch (Microsoft Internal)

- Use OneBranch template for all Microsoft-internal services
- Follow the standard OneBranch pipeline structure
- Use CloudTest integration for test reporting
- Leverage built-in compliance and security scanning

### Pipeline Structure

```yaml
trigger:
  branches:
    include: [main]
  paths:
    exclude: ['docs/**', '*.md']

pool:
  vmImage: 'ubuntu-latest'

stages:
- stage: Build
  jobs:
  - job: BuildAndTest
    steps:
    - task: Cache@2          # Cache dependencies
    - script: dotnet restore
    - script: dotnet build
    - script: dotnet test --collect:"XPlat Code Coverage"
    - task: PublishBuildArtifacts@1

- stage: DeployDev
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

- stage: DeployStaging
  dependsOn: DeployDev
  condition: succeeded()

- stage: DeployProd
  dependsOn: DeployStaging
  condition: succeeded()
  jobs:
  - deployment: Production
    environment: 'production'  # Has approval gate
    strategy:
      rolling:
        deploy:
          steps: [...]
```

### Azure Pipelines Best Practices

- **Use templates** for shared pipeline logic across repos
- **Stage dependencies** with `dependsOn` — make the flow explicit
- **Approval gates** on environment resources, not in pipeline YAML
- **Variable groups** per environment, linked to Key Vault
- **Conditional stages** — only deploy when building from main branch
- **Service connections** scoped to resource groups, not subscriptions

## GitHub Actions Specifics

### Workflow Structure

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read        # Least privilege — explicit permissions

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@<sha>
    - uses: actions/setup-node@<sha>
      with:
        cache: 'npm'    # Built-in caching
    - run: npm ci
    - run: npm test
    - run: npm run build

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: staging
    # ...

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production  # Has approval requirement
    # ...
```

### GitHub Actions Best Practices

- **Pin actions by SHA** — not by tag, not by branch. Tags can be moved, SHAs cannot
- **Use reusable workflows** for shared CI/CD patterns across repos
- **Matrix builds** for multi-platform/version testing
- **Explicit permissions** — set `permissions` at workflow and job level. Default to read-only
- **OIDC authentication** to Azure/AWS/GCP — no stored cloud credentials
- **Concurrency control** — prevent parallel deployments to the same environment:
  ```yaml
  concurrency:
    group: deploy-${{ github.ref }}
    cancel-in-progress: false
  ```
- **Cache aggressively** — `actions/cache` for dependencies, Docker layer caching for images

## Rollback

**Every deployment must have an automated rollback plan.**

### Rollback Triggers (Automated)

- Health check failure post-deployment
- Error rate exceeds threshold (> 1% 5xx for 5 minutes)
- Latency P95 exceeds 2x baseline
- Key business metric drops > 10%

### Rollback Methods

| Method | Speed | When |
|---|---|---|
| **Kubernetes rollout undo** | Seconds | Container deployments |
| **Blue-green traffic switch** | Instant | Blue-green deployments |
| **Canary traffic reroute** | Seconds | Canary deployments |
| **Feature flag disable** | Instant | Flag-gated features |
| **Redeploy previous artifact** | Minutes | Last resort |

### Rollback Rules

- Rollback is always preferable to forward-fixing in production under pressure
- Test rollback procedures regularly — an untested rollback is not a rollback plan
- Database rollback: Migrations must be backward-compatible to support rollback without data migration reversal
- Document rollback procedure in deployment runbook

## Pipeline Performance

- **Cache everything:** Dependencies, Docker layers, build outputs, test results
- **Parallel jobs:** Run independent stages/jobs concurrently. Don't serialize what can parallelize
- **Conditional stages:** Skip deployment stages on PRs. Skip unchanged projects in monorepos
- **Artifact retention:** Keep last 30 days for dev, 90 days for staging, 1 year for production
- **Self-hosted agents:** For builds that need large disk, specific hardware, or network access to private resources
- **Incremental builds:** Use build caching (Nx, Turborepo, BuildXL) for monorepos

## Pipeline Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|---|---|---|
| Secrets in YAML | Exposed in source control | Variable groups / GitHub Secrets |
| No branch protection | Broken code merges to main | Require CI pass + reviews |
| Manual deployments | Error-prone, unrepeatable | Automate everything |
| Skipping staging | Production-only testing | Always promote through environments |
| No rollback plan | Stuck with broken production | Automated rollback on health failure |
| Flaky tests ignored | Erode trust in CI | Fix or quarantine within 1 sprint |
| Monolithic pipeline | Slow, hard to debug | Modular stages and templates |
| Deployer = approver | No separation of duties | Different person must approve |
