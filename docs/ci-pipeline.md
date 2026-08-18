# CI/CD Pipeline — Golden Path Reusable Workflow

## Overview

The **Golden Path CI** is a reusable GitHub Actions workflow (`.github/workflows/golden-path-ci.yml`) that standardizes CI/CD checks for Node.js/Express services deployed to AWS ECS Fargate via Terraform. It is designed to be adopted by any new service team in the platform, with minimal configuration.

A service team adopts the golden path by creating a simple caller workflow (e.g., `.github/workflows/todo-service-ci.yml`) that invokes the reusable workflow with project-specific inputs and secrets.

## Golden Path Workflow Architecture

```
┌──────────────────────────────────────┐
│   Caller: todo-service-ci.yml        │
│   (push to main, pull_request)       │
└──────────────┬───────────────────────┘
               │
               ├─────────────────┐
               │ run_terraform_  │
               │ apply: false    │ (PR)
               │                 │
               ▼                 ▼
        ┌───────────────────────────────────────────────┐
        │   Reusable: golden-path-ci.yml                │
        │   (workflow_call)                             │
        └───────────────────────────────────────────────┘
               │
     ┌─────────┼─────────┬──────────────┬──────────────┬────────────┐
     │         │         │              │              │            │
     ▼         ▼         ▼              ▼              ▼            ▼
   lint      test   security-scan  terraform-plan  docker-build  terraform-apply  build-and-push
   (node)    (jest)  (checkov)      (terraform)    (docker)      (terraform)      (docker ECR)
     │         │         │ (PR)         │ (PR)        │ (if:PR)       │ (if:apply)     │ (if:apply)
     └─────────┴─────────┴─────────────┴─────────────┘              │              │
               │                                                      ▼              │
               │                                                  [depends on       │
               │                                                   terraform-plan]  │
               │                                                      │              ▼
               │                                                      │        [depends on
               │                                                      │         terraform-
               │                                                      │         apply]
               ▼                                                      ▼              ▼
        ✅ PR passes all checks, ready for merge      ✅ Deployed to ECS   ✅ Images pushed to ECR
```

---

## Jobs & Purpose

### 1. **lint** (Always Runs)

**What it does:**
- Installs Node.js dependencies
- Runs ESLint on `packages/backend` and `packages/frontend`

**Why it's required:**
- Catches syntax errors, style violations, and code quality issues early
- Prevents inconsistent code from entering the codebase
- Enforces team coding standards

**Triggers on:** Every push and every PR

---

### 2. **test** (Always Runs)

**What it does:**
- Installs Node.js dependencies
- Runs Jest on `packages/backend` with coverage reporting
- Appends test coverage summary (Lines, Branches, Functions, Statements %) to GitHub Step Summary

**Why it's required:**
- Validates application behavior (CRUD operations, API endpoints)
- Enforces minimum 80% code coverage (job fails if coverage drops below 80%)
- Provides visibility into test quality via GitHub Actions UI

**Triggers on:** Every push and every PR

**Failure condition:** Coverage < 80%

---

### 3. **security-scan** (Conditional: PR only)

**What it does:**
- Installs Checkov (IaC security scanner)
- Scans `infra/` directory for HIGH severity findings
- Fails if any HIGH findings are detected

**Why it's required:**
- Detects security misconfigurations in Terraform code (unencrypted storage, overly permissive policies, etc.)
- Prevents insecure infrastructure from reaching production
- Leverages policy-as-code to enforce compliance

**Triggers on:** When `run_terraform_plan: true` (typically on PRs)

**Failure condition:** Any HIGH severity Checkov violation

---

### 4. **terraform-plan** (Conditional: PR only)

**What it does:**
1. Checks out code
2. Configures AWS credentials via OIDC (no long-lived secrets)
3. Downloads and installs Terraform
4. Runs `terraform init` with S3 backend state key injection
5. Creates mock provider variables (vpc_id, subnet IDs) if not applying
6. Runs `terraform plan -out=tfplan` to generate an executable plan
7. Outputs plan summary to GitHub Step Summary
8. Uploads plan artifact for later apply step

**Why it's required:**
- Validates IaC syntax and variable types
- Shows team what infrastructure changes will occur before applying
- Enables code review of infrastructure changes (drift detection, resource changes)
- Supports zero-trust deployment (plan first, apply second)

**Triggers on:** When `run_terraform_plan: true` (typically on PRs and main push)

**Permissions:** `id-token: write` (for OIDC token exchange)

**AWS Configuration:** Uses OIDC federation (`aws-actions/configure-aws-credentials@v4`) to assume a temporary role, avoiding long-lived static credentials.

---

### 5. **docker-build** (Conditional: PR only)

**What it does:**
- Builds backend Docker image from `packages/backend/Dockerfile`
- Builds frontend Docker image from `packages/frontend/Dockerfile`
- Does NOT push images or authenticate with AWS (no credentials needed)

**Why it's required:**
- Validates Dockerfiles are syntactically correct
- Catches build-time errors (missing dependencies, bad COPY paths, etc.)
- Ensures services can be containerized before attempting deployment

**Triggers on:** PR events only (`if: github.event_name == 'pull_request'`)

**Dependencies:** Runs after `lint` and `test` complete successfully

---

### 6. **terraform-apply** (Conditional: Push to main only)

**What it does:**
1. Checks out code
2. Configures AWS credentials via OIDC
3. Removes mock provider flags from `main.tf` (switches to real OIDC auth)
4. Runs `terraform init` with real S3 backend
5. Downloads plan artifact from `terraform-plan` job
6. Executes `terraform apply tfplan` (deterministic, non-interactive)
7. Extracts Terraform outputs (Load Balancer URL) and posts to Step Summary

**Why it's required:**
- Applies infrastructure plan to AWS (creates/updates resources)
- Only runs on push to main (not on PRs—review required before production change)
- Uses OIDC to assume temporary AWS credentials (no static secrets in workflow)
- Exports deployment URL for team visibility

**Triggers on:** Push to `main` only (`run_terraform_apply: true` only on main)

**Permissions:** `id-token: write` (for OIDC token)

**Dependencies:** Runs after `terraform-plan` completes

---

### 7. **build-and-push** (Conditional: Push to main only)

**What it does:**
1. Checks out code
2. Configures AWS credentials via OIDC
3. Uses Terraform to query stack outputs (ECR repository URLs)
4. Logs into Amazon ECR
5. Builds backend image with tags: `{repo}:latest` and `{repo}:{git-sha}`
6. Pushes backend image to ECR
7. Builds frontend image with tags: `{repo}:latest` and `{repo}:{git-sha}`
8. Pushes frontend image to ECR
9. Triggers ECS service update (force new deployment)

**Why it's required:**
- Builds production-ready container images
- Pushes images to ECR (AWS's managed container registry)
- Maintains both mutable `latest` tag and immutable commit SHA tags (for rollback)
- Automatically updates ECS tasks with new images

**Triggers on:** Push to `main` only (`build_and_push: true` only on main)

**Permissions:** `id-token: write` (for OIDC token)

**Dependencies:** Runs after `terraform-apply` completes

---

## How to Adopt the Golden Path

### Minimum Configuration

New service teams adopt the golden path by creating a caller workflow in `.github/workflows/{service-name}-ci.yml`:

```yaml
name: {Service Name} CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

### Key Points

1. **Trigger:** Set to `push: [main]` and `pull_request` for PR validation + production deployment
2. **Permissions:** MUST declare `id-token: write` at the top level (caller must request OIDC token)
3. **node_version:** Default is "20" — change only if using a different Node version
4. **terraform_version:** Default is "1.7.0" — change only if your stack requires a specific version
5. **run_terraform_plan:** Set to `true` to enable IaC validation and planning on every PR
6. **run_terraform_apply:** Conditional on `push && main` to deploy only on merges
7. **build_and_push:** Conditional on `push && main` to push images only on merges
8. **aws_role_arn secret:** Pass the OIDC role ARN from repo secrets (see below)

---

## Required Checks & Why They Matter

| Check | Runs | Failure Mode | Why Required |
|-------|------|--------------|--------------|
| **lint** | Always | ESLint errors | Code quality, consistency |
| **test** | Always | Jest or coverage <80% | Functional correctness, regression prevention |
| **security-scan** | PR (if enabled) | Checkov HIGH findings | Prevent insecure IaC from reaching prod |
| **terraform-plan** | PR (if enabled) | Terraform error or plan failure | Validate IaC syntax, show infrastructure diffs |
| **docker-build** | PR only | Docker build failure | Ensure images build before pushing to ECR |
| **terraform-apply** | Push to main | Apply failure | Deploy approved infrastructure to AWS |
| **build-and-push** | Push to main | ECR push failure | Update ECS tasks with latest container images |

**PR workflow (on `pull_request`):**
- lint ✅ → test ✅ → docker-build → (optional) security-scan → (optional) terraform-plan
- All must pass before PR merge is allowed

**Main workflow (on `push` to `main`):**
- lint ✅ → test ✅ → docker-build → security-scan → terraform-plan → **terraform-apply** → **build-and-push**
- Runs end-to-end: infrastructure, then container images

---

## Configuring Secrets: AWS Role ARN for OIDC

### Prerequisites

Your GitHub repository must have an **AWS IAM role configured for OIDC federation** that permits:
- `sts:AssumeRoleWithWebIdentity` from GitHub's OIDC provider
- Policies: `AmazonEC2FullAccess`, `AmazonECS_FullAccess`, `AmazonEC2ContainerRegistryFullAccess`, `S3` and `Terraform` state access

### Setup

1. **Get the role ARN from your platform team:**
   ```
   arn:aws:iam::123456789012:role/github-actions-todo-service-oidc-role
   ```

2. **Add it as a repository secret:**
   - Go to **Settings → Secrets and variables → Actions**
   - Click **New repository secret**
   - **Name:** `AWS_ROLE_ARN`
   - **Value:** `arn:aws:iam::123456789012:role/github-actions-todo-service-oidc-role`
   - Click **Add secret**

3. **Verify in caller workflow:**
   ```yaml
   jobs:
     call-golden-path:
       ...
       secrets:
         aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
   ```

### How OIDC Works

When `terraform-plan`, `terraform-apply`, or `build-and-push` jobs run:

1. GitHub Actions automatically generates an OIDC token (JWT)
2. `aws-actions/configure-aws-credentials@v4` exchanges the token for temporary AWS credentials
3. Credentials are valid for 1 hour and automatically expire
4. **No long-lived access keys are stored in GitHub Secrets** (more secure)

### Why OIDC Over Static Keys

- ✅ **Zero static secrets** — no IAM access keys in GitHub
- ✅ **Short-lived credentials** — 1 hour TTL, automatic expiry
- ✅ **Audit trail** — CloudTrail logs show which GitHub run, commit, actor assumed the role
- ✅ **Revocation** — disable role to block all GitHub Actions at once
- ❌ **No rotation needed** — OIDC tokens are ephemeral

---

## Workflow Diagram: PR → Main → Deployed

```
Developer pushes to feature branch
              │
              ▼
         (1) Create PR
              │
              ▼
   ┌──────────────────────┐
   │   PR Checks Kick Off  │
   │  (golden-path-ci)    │
   └──────────────────────┘
              │
     ┌────────┼────────┐
     ▼        ▼        ▼
   lint    test   docker-build
     │        │        │
     └────────┼────────┘
              ▼
        ✅ All pass?
              │
              ├─── No ──→ ❌ PR blocked
              │
              └─── Yes
                    ▼
         (2) Peer review approved
              │
              ▼
         (3) Merge to main
              │
              ▼
   ┌──────────────────────────┐
   │  Main Deployment Kicks   │
   │    Off (golden-path)     │
   └──────────────────────────┘
              │
     ┌────────┼────────┬──────────┐
     ▼        ▼        ▼          ▼
   lint    test   security-scan  terraform-plan
     │        │        │          │
     └────────┴────────┴──────────┘
              ▼
        ✅ All pass?
              │
              ├─── No ──→ ❌ Deployment blocked
              │
              └─── Yes
                    ▼
            terraform-apply
         (VPC, ECS, ALB created)
              │
              ▼
           build-and-push
      (Images pushed to ECR)
              │
              ▼
        ECS tasks updated
              │
              ▼
        ✅ Service live on
         Load Balancer
```

---

## Troubleshooting

### `terraform-plan` fails with "AWS was not able to validate credentials"

**Cause:** OIDC role ARN is missing or incorrect  
**Solution:**
1. Check that `AWS_ROLE_ARN` secret is set in repository settings
2. Verify the role ARN format: `arn:aws:iam::ACCOUNT:role/ROLE_NAME`
3. Confirm the role has trust relationship with GitHub OIDC provider

### `terraform-plan` fails with "backend configuration error"

**Cause:** S3 state bucket does not exist or is not accessible  
**Solution:**
1. Verify S3 bucket `pe-labs-terraform-state` exists in `us-east-2`
2. Confirm bucket is accessible from the OIDC role's account
3. Check Terraform backend config in `infra/stacks/dev/main.tf` (S3 backend block is commented out for local dev)

### `docker-build` fails with "Dockerfile not found"

**Cause:** Path to Dockerfile is incorrect  
**Solution:**
1. Verify `packages/backend/Dockerfile` exists
2. Verify `packages/frontend/Dockerfile` exists
3. Check working directory in the build step

### `build-and-push` fails with "role not assumed"

**Cause:** `id-token: write` is not declared at caller level  
**Solution:**
1. Ensure **top-level** `permissions:` in `todo-service-ci.yml` includes `id-token: write`
2. Declaring it only in the reusable workflow is not sufficient (GitHub only grants token to workflows that explicitly request it)

### Coverage report not appended to Step Summary

**Cause:** Jest test fails before coverage is generated, or coverage file path is incorrect  
**Solution:**
1. Fix the Jest test failure first
2. Verify coverage JSON file path: `packages/backend/coverage/coverage-summary.json`
3. Check that Jest is run with `--coverage` flag

---

## Next Steps

1. **Adopt in your service:** Copy the caller workflow template to `.github/workflows/{service}-ci.yml`
2. **Configure secrets:** Add `AWS_ROLE_ARN` to repository settings
3. **Push and test:** Merge a PR to main and watch the workflow deploy your service
4. **Monitor:** Review GitHub Actions logs, step summaries, and Terraform plan outputs

For questions or issues, contact the Platform Engineering team.
