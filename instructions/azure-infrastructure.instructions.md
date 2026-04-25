---
description: "Azure infrastructure best practices for IaC with Bicep and Terraform, resource management, RBAC, Key Vault, tagging, and Azure Well-Architected Framework alignment."
applyTo: "**/*.bicep,**/*.tf,**/*.tfvars,**/infra/**,**/infrastructure/**"
---

# Azure Infrastructure Best Practices

## Infrastructure as Code Principles

- ALL infrastructure must be defined in code — no portal-only resources
- IaC must be idempotent — re-running produces the same result
- IaC lives in version control alongside application code
- Prefer declarative (Bicep/Terraform) over imperative (scripts)
- Use modules for reusability and consistency

## Bicep Best Practices

- Use Azure Verified Modules (AVM) as the foundation
- Use `@description()` decorator on all parameters and outputs
- Use `@allowed()` for constrained parameter values
- Use `@minLength()`, `@maxLength()`, `@minValue()`, `@maxValue()` for validation
- Use parameter files for environment-specific values (`.bicepparam`)
- Prefer `existing` keyword for referencing existing resources
- Name resources with a consistent naming convention: `{resourceType}-{workload}-{environment}-{region}-{instance}`
- Use `dependsOn` only when implicit dependencies aren't sufficient

## Terraform Best Practices

- Use official Azure Verified Modules from the Terraform registry
- Use remote state (Azure Storage backend with state locking)
- Use workspaces or separate state files per environment
- Pin provider and module versions (`~> 3.0` not `>= 3.0`)
- Use `terraform plan` output in PR comments for review
- Structure: `main.tf`, `variables.tf`, `outputs.tf`, `providers.tf`, `locals.tf`
- Use `for_each` over `count` for named resource instances
- Use `moved` blocks for state refactoring (never manual state manipulation)

## Resource Management

- Use resource groups to group related resources by lifecycle
- Tag ALL resources:
  - `environment`: dev, staging, prod
  - `owner`: team or individual responsible
  - `cost-center`: for billing allocation
  - `project`: project/product name
  - `managed-by`: terraform, bicep, or manual
- Enable diagnostic settings and logging on all resources
- Use Azure Policy for governance enforcement
- Enable soft-delete on storage and Key Vault

## Security

- Use Managed Identities over service principals where possible
- Store ALL secrets in Key Vault — never in code, config files, or environment variables
- Use RBAC with principle of least privilege
- Use Private Endpoints for PaaS services (no public endpoints in production)
- Enable Azure Defender / Microsoft Defender for Cloud
- Enable audit logging on all resources
- Use network security groups with explicit deny rules
- Encrypt data at rest (enabled by default, verify customer-managed keys for sensitive data)
- Regular access reviews for RBAC assignments

## Environment Strategy

- Minimum 3 environments: dev, staging, production
- Production must match staging configuration exactly
- Use feature flags for gradual rollouts — don't branch infrastructure
- Dev can use lower SKUs; staging and prod must match SKUs

## Cost Management

- Use Azure Cost Management + Budgets with alerts
- Right-size VMs and databases (review monthly)
- Use Reserved Instances for stable workloads
- Auto-scale where possible
- Tag resources for cost allocation
- Clean up unused resources (orphaned disks, NICs, IPs)

## CI/CD for Infrastructure

- Validate IaC in PRs (`bicep build`, `terraform validate`, `terraform plan`)
- Use separate pipelines for infrastructure and application deployment
- Apply infrastructure changes before application deployment
- Always plan before apply — never auto-apply without review
- Include rollback plan for every infrastructure change
- Use deployment slots for zero-downtime deployments
