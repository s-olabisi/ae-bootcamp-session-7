# CI/CD Pipeline — Golden Path Workflow

This document explains the reusable CI/CD pipeline (`golden-path-ci.yml`) and how to adopt it for a new service.

## Overview

The **Golden Path CI workflow** is a reusable GitHub Actions pipeline that automates:

- **Code quality** — linting and testing
- **Infrastructure validation** — Terraform plan and security scanning
- **Container builds** — Docker image validation and push to ECR
- **Infrastructure deployment** — Terraform apply and ECS service updates

The workflow is designed to be **service-agnostic**: a new team adds a single caller workflow (e.g., `todo-service-ci.yml`) that invokes the golden path with their specific inputs and secrets.

## Golden Path Workflow Structure (`golden-path-ci.yml`)

### Trigger

```yaml
on:
  workflow_call:
    inputs:
      node_version: { default: "20" }
      terraform_version: { default: "1.7.0" }
      run_terraform_plan: { default: false }
      run_terraform_apply: { default: false }
      build_and_push: { default: false }
    secrets:
      aws_role_arn:
        description: AWS IAM role ARN for OIDC federation
        required: false
```

This is a **reusable workflow** — it must be invoked by a caller workflow (e.g., `todo-service-ci.yml`), not triggered directly. The inputs and secrets allow the caller to customize behavior without modifying the golden path.

### Jobs and Their Purpose

#### 1. **`lint`** (Always runs)

**What it validates:**

- ESLint on backend (`packages/backend`)
- ESLint on frontend (`packages/frontend`)

**Why it is required:**

- Catches syntax errors, unused variables, and style violations early
- Prevents inconsistent code from reaching main branch
- Fast feedback loop (< 1 minute)

**Example failure scenario:** A developer forgets a semicolon or uses an undefined variable; the workflow catches this before review.

---

#### 2. **`test`** (Always runs)

**What it validates:**

- Jest test suite on backend
- **Code coverage must be ≥ 80%** (enforced in this job)
- Outputs coverage summary to pull request

**Why it is required:**

- Ensures code behaves as expected before deployment
- 80% coverage threshold prevents untested logic from reaching production
- Coverage report is visible in the PR, encouraging high-quality tests

**Example failure scenario:** A new todo endpoint lacks tests; CI blocks the PR until tests are added and coverage reaches 80%.

---

#### 3. **`security-scan`** (Conditional: `run_terraform_plan == true`)

**What it validates:**

- Runs `checkov` on entire `infra/` directory
- **Fails if ANY HIGH-severity security issues are found**

**Why it is required:**

- Prevents misconfigurations (e.g., public S3 buckets, unencrypted databases)
- Checkov scans for AWS best practices, compliance (CIS, PCI-DSS), and common IaC mistakes
- Blocks insecure infrastructure from being deployed

**Example failure scenario:** A developer sets `publicly_accessible = true` on an RDS database. Checkov flags this HIGH-severity issue; CI fails until it is corrected.

---

#### 4. **`terraform-plan`** (Conditional: `run_terraform_plan == true`)

**What it validates:**

- Runs `terraform validate` (syntax and required arguments)
- Runs `terraform plan` against `infra/stacks/dev`
- Authenticates to AWS via OIDC (no long-lived credentials)
- On pull requests, mocks AWS resources (uses `vpc-mock`, `subnet-mock`) to validate syntax without AWS access
- On push to main, uses real AWS state to show actual deployment changes
- Uploads plan artifact for `terraform-apply` job

**Why it is required:**

- `terraform validate` catches typos and missing variables early
- `terraform plan` shows what will be created/modified/deleted before apply
- OIDC reduces credential sprawl and improves auditability
- Plan artifacts ensure consistency between plan and apply

**Example failure scenario:** A developer defines a variable but never assigns it. `terraform plan` fails with a clear error message before the apply job runs.

---

#### 5. **`docker-build`** (Conditional: `github.event_name == 'pull_request'`)

**What it validates:**

- Builds backend Dockerfile (no push)
- Builds frontend Dockerfile (no push)
- Ensures Dockerfile syntax is correct and all dependencies are available

**Why it is required:**

- Catches Dockerfile errors (e.g., missing RUN commands, broken COPY paths) before merge
- No AWS credentials needed for PR builds (faster feedback)
- Only runs on PRs to avoid redundant builds on main (main's build happens in `build-and-push` job)

**Example failure scenario:** A developer updates `packages/backend/package.json` but forgets to rebuild the Docker image; the workflow fails, alerting them to update the Dockerfile.

---

#### 6. **`terraform-apply`** (Conditional: `run_terraform_apply == true`, needs `terraform-plan`)

**What it does:**

- Downloads the `terraform-plan` artifact
- Authenticates to AWS via OIDC
- Runs `terraform apply` to provision/update infrastructure in dev
- Outputs the ALB service URL to the PR/commit

**Why it is required:**

- Infrastructure changes must be applied consistently (no manual `terraform apply` commands)
- OIDC authentication ensures only GitHub Actions (and authorized branch pushes) can deploy
- Only runs on push to `main`, not on PRs (safe by design)

**Example success scenario:** A developer merges a PR that scales ECS task CPU. The workflow automatically applies the change and outputs the updated service URL.

---

#### 7. **`build-and-push`** (Conditional: `build_and_push == true`, needs `terraform-apply`)

**What it does:**

- Queries Terraform state to get ECR repository URLs
- Builds backend and frontend Docker images
- Tags images with both `git-sha` (immutable) and `latest`
- Pushes images to ECR
- Triggers ECS service update to roll out new images

**Why it is required:**

- Container images must be versioned and pushed before ECS deployment
- SHA-tagged images provide audit trail (can revert to any previous commit)
- `latest` tag simplifies local development
- ECS service update ensures new code actually runs in production

**Example scenario:** A developer pushes a fix to main. The workflow builds images, pushes them to ECR, and ECS rolls out the updated service within 5 minutes.

---

## How to Adopt: Minimum Caller Workflow

To adopt the golden path for a new service, create a **caller workflow** that invokes `golden-path-ci.yml`. Here is the minimum required `todo-service-ci.yml`:

```yaml
name: Todo Service CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write # ← CRITICAL: Required for OIDC; GitHub only grants token to workflows that declare this

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

### Key Design Decisions

| Feature                                                                                          | Rationale                                                                                |
| ------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| **`permissions: id-token: write`**                                                               | Enables OIDC token retrieval; must be declared at caller level, not in reusable workflow |
| **`run_terraform_plan: true`**                                                                   | Always validate IaC (security and correctness)                                           |
| **`run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}`** | Only apply infrastructure on merges to main; PRs are dry-runs only                       |
| **`build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}`**      | Only push images on main; PRs validate Dockerfiles but don't push                        |
| **`secrets.aws_role_arn`**                                                                       | Passed as a workflow secret (stored in repository settings); never hardcoded             |

---

## Required Checks Summary

| Check            | Runs                    | Validates                  | Blocks PR if Failed? |
| ---------------- | ----------------------- | -------------------------- | -------------------- |
| `lint`           | Always                  | ESLint on backend/frontend | Yes                  |
| `test`           | Always                  | Jest ≥80% coverage         | Yes                  |
| `security-scan`  | On `run_terraform_plan` | Checkov HIGH findings      | Yes                  |
| `terraform-plan` | On `run_terraform_plan` | Terraform syntax & plan    | Yes                  |
| `docker-build`   | PRs only                | Dockerfile builds          | Yes                  |

All five checks must pass before a PR can be merged to `main`.

---

## Configuring Secrets: OIDC Role ARN

### Overview

The `terraform-plan` and `terraform-apply` jobs authenticate to AWS via **OIDC (OpenID Connect)**, which is more secure than long-lived access keys:

- No AWS credentials stored in GitHub
- Each GitHub Actions job gets a short-lived, session-scoped token
- Token is valid only for the specific repository and branch
- Audit trail in AWS CloudTrail shows "service: sts.amazonaws.com" calls from GitHub

### Step-by-Step Configuration

#### 1. **Create an IAM Role in AWS**

The role must trust GitHub Actions as an OIDC provider. Use these AWS-provided values:

```terraform
data "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
}

resource "aws_iam_role" "github_actions" {
  name = "todo-service-github-actions"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = data.aws_iam_openid_connect_provider.github.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            "token.actions.githubusercontent.com:sub" = "repo:s-olabski/ae-bootcamp-session-7:*"
          }
        }
      }
    ]
  })
}

# Attach permissions for Terraform to manage ECS, ECR, networking, etc.
resource "aws_iam_role_policy_attachment" "terraform" {
  role       = aws_iam_role.github_actions.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"  # Adjust to least privilege
}
```

**Get the role ARN:**

```bash
aws iam get-role --role-name todo-service-github-actions --query 'Role.Arn' --output text
# Output: arn:aws:iam::123456789012:role/todo-service-github-actions
```

#### 2. **Store Role ARN as a GitHub Secret**

Navigate to **Settings → Secrets and variables → Actions** and create a new secret:

| Field     | Value                                                        |
| --------- | ------------------------------------------------------------ |
| **Name**  | `AWS_ROLE_ARN`                                               |
| **Value** | `arn:aws:iam::123456789012:role/todo-service-github-actions` |

#### 3. **Verify in Workflow**

The caller workflow passes the secret to the reusable workflow:

```yaml
jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }} # ← Retrieved from repository secrets
```

The `terraform-plan` job uses it:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.aws_role_arn }} # ← OIDC federation
    aws-region: us-east-1
```

#### 4. **Test the Integration**

Push a commit to a PR and check the `terraform-plan` job logs:

```
[Configure AWS credentials] $AWS_ACCOUNT_ID=123456789012
[Configure AWS credentials] $AWS_ROLE_ARN=arn:aws:iam::123456789012:role/todo-service-github-actions
```

If you see this, OIDC is working correctly. If the job fails with a permissions error, check:

- [ ] Role ARN is correct and copied to GitHub secrets
- [ ] IAM role trust policy includes `token.actions.githubusercontent.com`
- [ ] Repository slug in the trust policy matches your repo (e.g., `s-olabski/ae-bootcamp-session-7`)

---

## Example PR Flow

Here's what happens when a developer opens a PR to `main`:

```
1. Push feature branch
   ↓
2. GitHub Actions triggers: Pull Request event
   ↓
3. todo-service-ci.yml runs with:
   - run_terraform_plan: true ← IaC validation
   - run_terraform_apply: false ← No real AWS changes
   - build_and_push: false ← No ECR push
   ↓
4. Jobs run in parallel (lint, test, docker-build)
   ↓
5. terraform-plan runs (after lint/test pass)
   - Uses AWS OIDC to assume role
   - Mocks VPC/subnet IDs (does not touch real AWS)
   - Shows plan output in PR comment
   ↓
6. All checks pass → PR is ready to review
   ↓
7. Developer merges PR to main
   ↓
8. GitHub Actions triggers: Push event
   ↓
9. todo-service-ci.yml runs with:
   - run_terraform_plan: true
   - run_terraform_apply: true ← Real infrastructure deployment
   - build_and_push: true ← Push images to ECR
   ↓
10. terraform-plan runs (plan is uploaded as artifact)
    ↓
11. terraform-apply runs (downloads and applies the plan)
    - Queries Terraform outputs for ECR repository URLs
    ↓
12. build-and-push runs
    - Pushes images to ECR with SHA and latest tags
    - Triggers ECS service update
    ↓
13. Service is updated in dev environment (within 5 minutes)
```

---

## Troubleshooting

### Issue: `terraform-plan` fails with "Backend initialization required"

**Cause:** You ran `terraform init -reconfigure` locally, which created a `main.tf.lock` file.

**Solution:** Ensure `infra/stacks/dev/main.tf` has no lock files; the CI pipeline will initialize the backend correctly.

### Issue: `terraform-apply` fails; `terraform-plan` passed

**Cause:** Infrastructure state changed between plan and apply (e.g., another developer deployed).

**Solution:** The workflow re-runs `terraform plan` to verify consistency. This is a safety feature; if unsafe changes are detected, the apply is skipped.

### Issue: `test` job fails with "Coverage below 80%"

**Cause:** New code lacks test coverage.

**Solution:** Add tests to `packages/backend/__tests__/` until coverage reaches 80%.

---

## Next Steps

- **Local testing:** Run `npm lint`, `npm test`, `terraform validate` locally before pushing
- **Customization:** Adjust the caller workflow inputs for different services (different Node versions, Terraform versions, etc.)
- **Monitoring:** Check CloudWatch Logs for ECS service updates after a successful deploy
