# DevOps Interview Preparation Guide
### Git · GitHub Actions · Terraform
#### For Engineers with 4+ Years Experience (2026)

---

## How to Use This Guide

Each section follows this structure:

1. **Concept** — what it is and why it matters
2. **Practice Activity** — hands-on task using the zen-pharma project
3. **Interview Questions & Answers** — what interviewers ask at this level

**The golden rule for answering at senior level:**
> Situation → Decision → Trade-off → Outcome

Every answer should explain *why* you made a choice, not just *what* you did.
Generic answers get filtered out. Real project examples get you hired.

---

---

# PART 1: GIT

---

## 1.1 Branching Strategies

### Concept

A branching strategy defines how engineers collaborate on code through git branches. The two most common in 2026:

| Strategy | How it works | Best for |
|---|---|---|
| **Trunk-based development** | Short-lived feature branches, merge to main frequently | Infrastructure, platform teams, CI/CD-heavy projects |
| **GitFlow** | develop → release → main with long-lived branches | Products with scheduled releases, mobile apps |

For infrastructure code (Terraform), trunk-based is almost always the right choice. Long-lived branches cause state drift and merge conflicts that are painful to resolve.

### Practice Activity

```bash
# 1. From your zen-infra repo, create a feature branch
git checkout -b feature/test-branching

# 2. Make a small change (add a comment to envs/dev/main.tf)
# 3. Commit it
git add envs/dev/main.tf
git commit -m "test: add comment to dev main.tf"

# 4. Push and open a PR on GitHub
git push origin feature/test-branching

# 5. After merging, visualise the history
git log --oneline --graph --all

# 6. Try squash merge via GitHub UI (Merge dropdown → Squash and merge)
# Observe how the feature branch commits become a single commit on main
```

**What to observe:**
- With regular merge: you see a merge commit and all feature branch commits in history
- With squash merge: you see one clean commit on main — "test: add comment to dev main.tf"
- `git log --graph` shows the branch lines visually

---

### Interview Questions

---

**Q1: What branching strategy would you use for a team of 10 engineers working on infrastructure?**

**A:** For infrastructure code I prefer **trunk-based development** with short-lived feature branches. Engineers create branches like `feature/add-rds-module`, make changes, open a PR, get it reviewed, and merge to main — ideally within 1–2 days. Long-lived branches cause merge conflicts and drift between what is in code and what is in AWS.

For application code with multiple environments, some teams use GitFlow (develop → staging → main), but the overhead of maintaining multiple long-lived branches often outweighs the benefit. Trunk-based with feature flags or environment promotion via directory structure is cleaner.

*Project tie-in: In zen-infra we use trunk-based — feature branches for changes, PRs to main, no long-lived dev/qa/prod branches. Environment separation is handled by directory structure (envs/dev, envs/qa, envs/prod), not by branches.*

---

**Q2: What is the difference between merge, squash merge, and rebase? When would you use each?**

**A:**
- **Merge** — creates a merge commit preserving full branch history. Good when you want traceability of exactly what happened on the feature branch. Creates non-linear history.
- **Squash merge** — combines all commits on the feature branch into one commit on main. Clean linear history but you lose granular commit detail from the branch. Good for feature branches with many "WIP" or "fix typo" commits.
- **Rebase** — replays your commits on top of main, linear history without a merge commit. Looks like the work was done directly on main. **Never rebase shared or public branches** — it rewrites history and breaks other people's work.

For infrastructure repos I prefer squash merge. A feature like "add RDS module" should appear as one clean commit on main, not 8 commits of "try again", "actually fix it", "forgot variable".

---

**Q3: How do you handle a merge conflict in a Terraform file?**

**A:** Never resolve Terraform conflicts by just picking one side. Steps:

1. `git status` to identify conflicted files
2. Open the file — understand what each side changed (Terraform is declarative, so conflicts are usually about resource attributes or variable values)
3. Manually merge the intent of both changes
4. Run `terraform validate` to check syntax
5. Run `terraform plan` to verify the resolved file produces the expected changes
6. Commit the resolution

The key mistake engineers make is resolving conflicts without running a plan. The merged result might be syntactically valid but semantically wrong — for example, two engineers both modified the same security group rule in different ways and the merge removed one of the rules silently.

---

**Q4: What is `git cherry-pick` and when would you use it?**

**A:** Cherry-pick applies a specific commit from one branch onto another without merging the entire branch.

Real scenario: You have a critical Terraform fix on a feature branch that is not reviewed yet, but production is broken and needs that fix now. You cherry-pick just that commit to a hotfix branch, apply it to prod, then continue the full review process for the feature branch separately.

The risk is that cherry-picked commits can create duplicate commits when the feature branch eventually merges, causing conflicts. Use it sparingly — only for genuine hotfixes.

---

**Q5: What is `git reflog` and when has it saved you?**

**A:** `git reflog` records every movement of HEAD — including commits that are no longer reachable from any branch. It is the undo button for git operations that seem irreversible.

Scenario: You ran `git reset --hard HEAD~3` to undo 3 commits and then realised you needed one of them. `git log` will not show them anymore, but `git reflog` will — you can see the SHA of the lost commit and `git cherry-pick` it back, or `git reset --hard` to it.

For DevOps engineers this is valuable when someone accidentally force-pushes and overwrites commits. `reflog` on their local machine can recover them.

---

## 1.2 Git Hooks

### Concept

Git hooks are scripts that run automatically at certain points in the git workflow — `pre-commit`, `pre-push`, `post-merge` etc. They live in `.git/hooks/` which is **not tracked by git**, making enforcement across a team a challenge.

Tools that solve this:
- **pre-commit framework** (`pre-commit.com`) — hooks defined in `.pre-commit-config.yaml`, engineers run `pre-commit install` once
- **Husky** (for Node projects) — hooks in `package.json`, installed via `npm install`

### Practice Activity

```bash
# In zen-infra, create a pre-commit hook that runs terraform fmt
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
echo "Running terraform fmt check..."
cd envs/dev && terraform fmt -check -recursive
if [ $? -ne 0 ]; then
  echo "Terraform formatting issues found. Run 'terraform fmt' to fix."
  exit 1
fi
EOF

chmod +x .git/hooks/pre-commit

# Now try committing a badly formatted tf file
# Add extra spaces to envs/dev/main.tf and try to commit
# The hook should block it

# Fix with:
cd envs/dev && terraform fmt
```

**What to observe:**
- The hook runs before every commit
- It blocks the commit if formatting is wrong
- `.git/hooks/` is not committed — if a teammate clones the repo, they don't get this hook automatically

---

### Interview Questions

---

**Q6: What are git hooks and how do you enforce them across a team?**

**A:** Git hooks are scripts that run automatically at points in the git lifecycle. The problem is `.git/hooks/` is not tracked by git, so every engineer must set them up manually.

Teams solve this with the **pre-commit framework** — you define hooks in `.pre-commit-config.yaml` at the repo root (this IS tracked by git), and engineers run `pre-commit install` once after cloning. From then on the hooks run automatically.

For a Terraform repo, useful pre-commit hooks:
- `terraform fmt` — enforce formatting before commit
- `terraform validate` — catch syntax errors locally
- `detect-secrets` — prevent accidental credential commits
- `trailing-whitespace` — keep files clean

The enforcement mechanism is that CI also runs these same checks. If an engineer skips the local hook (`git commit --no-verify`), CI will catch it and block the PR.

---

---

# PART 2: GITHUB ACTIONS

---

## 2.1 Workflow Triggers

### Concept

Triggers (`on:`) define when a workflow runs. The most important triggers for CI/CD:

| Trigger | When it fires |
|---|---|
| `push` | Code lands on the branch (direct push or merge) |
| `pull_request` | PR opened, updated, or reopened targeting the branch |
| `workflow_dispatch` | Manual trigger from the GitHub UI with optional inputs |
| `schedule` | Cron schedule (e.g., nightly drift detection) |
| `workflow_call` | Called by another workflow (reusable workflows) |

### Practice Activity

```bash
# Test 1 — paths filter
# Make a change only to README.md and push to main
# Observe: pipeline does NOT trigger (README not in paths filter)

# Test 2 — terraform file change
# Make a change to envs/dev/main.tf and push
# Observe: pipeline DOES trigger

# Test 3 — manual dispatch
# Go to Actions → Terraform Infrastructure → Run workflow
# Select action: plan
# Observe: plan runs without any code change

# Test 4 — read the context
# Add this step temporarily to the plan job:
#   - name: Debug context
#     run: |
#       echo "Event: ${{ github.event_name }}"
#       echo "Ref: ${{ github.ref }}"
#       echo "Actor: ${{ github.actor }}"
# Push and read the output in the Actions logs
```

---

### Interview Questions

---

**Q7: How do you prevent a pipeline from running on every commit when only documentation changed?**

**A:** Use `paths` filters on the trigger:

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'modules/**'
```

Changes to `README.md`, `.gitignore`, or `docs/` will not trigger the pipeline. This saves CI minutes and reduces noise.

*Project tie-in: In zen-infra the pipeline only triggers when `envs/dev/**` or `modules/**` change. Committing workflow file changes or documentation does not accidentally run terraform apply.*

---

**Q8: What is the difference between `push` and `pull_request` triggers? Why do you need both?**

**A:**
- `push` fires when commits land on the branch (including merges)
- `pull_request` fires when a PR is opened, updated, or synchronized against the target branch

You need both because they serve different purposes. On a PR you want to run checks and post feedback (plan output as a comment) but NOT apply. On a push to main (after merge) you want to actually apply the changes.

If you only had `push`, PRs would get no feedback until after merge — too late to catch problems. If you only had `pull_request`, the actual apply after merge would never happen.

---

**Q9: What is `concurrency` in GitHub Actions and why is it critical for Terraform pipelines?**

**A:** Concurrency controls how many workflow runs can execute simultaneously for a given group.

For Terraform this is critical because two simultaneous `terraform apply` runs against the same state file will corrupt state. Even with S3 state locking, the second run fails with a lock error and may leave state in a bad condition.

```yaml
concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false
```

`cancel-in-progress: false` means the second run waits for the first to finish rather than cancelling it. For Terraform you always want to wait — cancelling a running apply is dangerous and leaves partial infrastructure state.

---

## 2.2 Jobs, Steps, and Dependencies

### Concept

A workflow contains **jobs**. Jobs contain **steps**. Key properties:

| Property | What it does |
|---|---|
| `runs-on` | Specifies the VM type — fresh machine per job |
| `needs` | Defines job execution order and dependencies |
| `if` | Condition for whether a job or step runs |
| `environment` | Links to a GitHub Environment — enables approval gates and environment secrets |
| `continue-on-error` | If true, job continues even if this step fails |

### Practice Activity

```bash
# 1. Open zen-infra/.github/workflows/terraform.yml
# 2. Trace the job dependency chain:
#    plan → apply (needs: plan)
#    plan → destroy (independent)

# 3. Add a temporary notify job to see job chaining in action:
#
# notify:
#   name: Notify Complete
#   runs-on: ubuntu-latest
#   needs: apply
#   if: always()
#   steps:
#     - run: echo "Apply finished with status ${{ needs.apply.result }}"
#
# Push to a feature branch, open a PR, observe the 3-job chain in Actions UI
# Remove the notify job after testing
```

---

### Interview Questions

---

**Q10: What is the difference between `needs` and `if` in GitHub Actions jobs?**

**A:**
- `needs` defines **execution order** — a job will not start until all jobs listed in `needs` have completed. Also gives access to outputs from those jobs.
- `if` defines **whether a job runs at all** — a conditional evaluated before the job starts.

They work together:

```yaml
apply:
  needs: plan          # wait for plan to finish first
  if: github.ref == 'refs/heads/main'   # but only run on main branch
```

Without `needs`, all jobs run in parallel by default. Without `if`, jobs always run regardless of context.

*Project tie-in: The apply job `needs: plan` so it waits for plan to succeed and upload the tfplan artifact. The `if` condition ensures apply only runs on main push or manual dispatch, not on PRs.*

---

**Q11: What are GitHub Actions artifacts and when would you use them instead of re-running a command?**

**A:** Artifacts are files uploaded from one job and downloaded by another job in the same workflow run. Jobs run on separate VMs so they do not share a filesystem — artifacts bridge that gap.

For Terraform specifically: the plan job runs `terraform plan -out=tfplan` and uploads the binary plan file as an artifact. The apply job downloads that exact file and runs `terraform apply tfplan`. This guarantees apply executes **exactly what you reviewed** — if you re-ran `terraform plan` in the apply job instead, infrastructure state might have changed between plan and apply, causing drift between what was approved and what gets applied.

Artifacts expire (1 day retention in this project) — they are not for long-term storage, just for passing data between jobs in the same run.

---

**Q12: What is `continue-on-error` and when should you use it carefully?**

**A:** `continue-on-error: true` on a step means if that step fails, the job continues to the next step rather than stopping. The step is still marked failed in the UI but does not block subsequent steps.

Use it when:
- You want to collect output regardless of success or failure (terraform plan — you want to post the plan comment even if plan fails)
- A check is informational, not blocking (terraform fmt — report it but do not block the pipeline)

Use it carefully because it hides failures. The pattern in this project: use `continue-on-error: true` on the plan step, then have a separate "Check plan status" step that reads `steps.plan.outcome` and runs `exit 1` if it failed. This lets the PR comment get posted before the job fails.

---

## 2.3 Secrets and Environments

### Concept

GitHub has three levels of secrets:

| Level | Scope | Approval gate |
|---|---|---|
| **Repository secrets** | All jobs in the repo | No |
| **Environment secrets** | Only jobs referencing that environment | Optional (via protection rules) |
| **Organisation secrets** | All repos in the org | No |

GitHub **Environments** are a separate feature that provides:
- Required reviewer approval gates
- Deployment branch restrictions
- Their own set of secrets

### Practice Activity

```bash
# 1. Go to github.com/ravdy/zen-infra → Settings → Environments → dev
# 2. Review the protection rules — required reviewers, wait timer
# 3. Trigger a workflow_dispatch with action: apply
# 4. Observe the apply job pause and show "Review deployments" button
# 5. Approve it and watch apply proceed
# 6. Try triggering with action: destroy but leave confirm_destroy blank
#    Observe: destroy job is skipped silently (if condition not met)
```

---

### Interview Questions

---

**Q13: What is the difference between repository secrets and environment secrets? When do you use each?**

**A:**
- **Repository secrets** — available to all jobs in all workflows in the repo. No approval gate.
- **Environment secrets** — available only to jobs that specify that environment. Can have protection rules (required reviewers, deployment branches).

Decision logic:
- Secrets the same across all environments (AWS credentials, Slack webhook) → repository secrets
- Secrets that differ per environment (DB passwords, JWT secrets) AND you want an approval gate → environment secrets
- Secrets that differ per environment but no approval gate needed → repository secrets with prefixes (`DEV_DB_PASSWORD`, `QA_DB_PASSWORD`)

*Project tie-in: AWS credentials are repository secrets — same key used across all environments. DB passwords use prefixed repository secrets (`DEV_DB_PASSWORD`) for the plan job which has no approval gate, and the `environment: dev` on the apply job provides the approval gate.*

---

**Q14: How do you share a workflow step across multiple workflows without duplicating code?**

**A:** Use **reusable workflows**. Define a workflow with `on: workflow_call`, expose inputs and secrets, and call it from other workflows with `uses:`.

```yaml
# _reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      AWS_ACCESS_KEY_ID:
        required: true

jobs:
  deploy:
    steps:
      - run: echo "Deploying to ${{ inputs.environment }}"
```

```yaml
# service-ci.yml
jobs:
  deploy:
    uses: ./.github/workflows/_reusable-deploy.yml
    with:
      environment: dev
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
```

*Project tie-in: zen-pharma-backend has a `_reusable-update-gitops.yml` workflow called by all 4 microservice CI pipelines to update the GitOps repo with the new image tag. Without it, that logic would be duplicated in 4 workflow files and any change would need updating in 4 places.*

---

## 2.4 Debugging and Troubleshooting

### Concept

When a pipeline fails, the error is rarely on the last line. The pattern:
1. Read the raw logs — expand the step, scroll UP from the bottom
2. Enable debug mode — add secret `ACTIONS_STEP_DEBUG = true`
3. Add echo statements to print context values
4. Reproduce locally with the same environment

### Practice Activity

```bash
# 1. Intentionally break terraform.yml — add a step with an invalid command:
#    - name: Bad step
#      run: notacommand --invalid
# Push to a feature branch and read the failure in Actions

# 2. Add debug echoes to the plan job:
#    - name: Debug
#      run: |
#        echo "Event name: ${{ github.event_name }}"
#        echo "Ref: ${{ github.ref }}"
#        echo "Actor: ${{ github.actor }}"
#        echo "SHA: ${{ github.sha }}"
# Push and read the output

# 3. Add ACTIONS_STEP_DEBUG = true as a repository secret
# Re-run a workflow and observe the verbose debug output
# Remove the secret after testing

# 4. Revert all changes after these experiments
```

---

### Interview Questions

---

**Q15: A GitHub Actions job is failing but the error message is unclear. How do you debug it?**

**A:** Layered approach:

1. **Read the raw logs** — expand the failing step in the Actions UI, scroll up from the bottom. GitHub shows the last few lines prominently but the actual error is usually higher up.
2. **Enable debug logging** — add repository secret `ACTIONS_STEP_DEBUG = true`. GitHub prints verbose debug output for every step including internal runner operations.
3. **Add echo statements** — print context variables to understand what values are being passed: `echo "Event: ${{ github.event_name }}"`, `echo "Ref: ${{ github.ref }}"`.
4. **Check `if` conditions** — a job silently skipped because an `if` evaluated to false looks like a bug but is not. Add a debug step before it to print the condition values.
5. **Reproduce locally** — for Terraform, run `terraform plan` locally with the same variables. For shell scripts, run them in a Docker container matching `ubuntu-latest`.

---

**Q16: You merged a PR and the apply job was skipped silently. No error, just skipped. What do you check?**

**A:** Silent skips in GitHub Actions are always caused by an `if` condition evaluating to false. Checklist:

1. Check the job's `if` condition — for the apply job in this project: `github.ref == 'refs/heads/main' && github.event_name == 'push'`. If this was a `workflow_dispatch` and not a `push`, the condition is false.
2. Check `needs` — if a dependency job failed, the downstream job is skipped.
3. Check the `paths` filter on the trigger — if no matching files changed, the entire workflow is skipped.
4. Add a debug step that always runs (`if: always()`) to print context values.

*Project tie-in: This exact issue occurred — the apply job was silently skipped when triggered via `workflow_dispatch` because the `if` only allowed `push` events. The fix was adding `|| (github.event_name == 'workflow_dispatch' && github.event.inputs.action == 'apply')` to the condition.*

---

---

# PART 3: TERRAFORM

---

## 3.1 State Management

### Concept

Terraform state is a JSON file mapping your configuration (what you declared) to real infrastructure (what exists in AWS). It tracks resource IDs, attributes, and dependencies.

Remote state (S3) solves:
1. **Team access** — multiple engineers and CI/CD pipelines share the same state
2. **State locking** — prevents two simultaneous applies from corrupting state
3. **Durability** — state survives machine restarts and CI runner destruction

### Practice Activity

```bash
cd zen-infra/envs/dev

# 1. List all resources in state
terraform state list

# 2. Inspect a specific resource in detail
terraform state show module.ecr.aws_ecr_repository.main[\"api-gateway\"]

# 3. See the full current state
terraform show

# 4. Simulate drift detection:
#    - Go to AWS Console → ECR → open api-gateway repo → edit tags (add a tag manually)
#    - Run terraform plan
#    - Observe: Terraform detects the tag drift and plans to remove it

# 5. Check the S3 backend
#    - Go to AWS Console → S3 → zen-pharma-terraform-state-ravdy
#    - Navigate to envs/dev/terraform.tfstate
#    - Download and open it — see the raw JSON state structure
```

---

### Interview Questions

---

**Q17: What is Terraform state and why is it stored remotely?**

**A:** Terraform state is a JSON file that maps your configuration to real infrastructure. Without it, Terraform has no way to know that `aws_vpc.main` in your code is the specific VPC `vpc-0abc123` in AWS — it would try to create a new one every time.

Remote state (S3 in this case) solves three problems:
1. **Team access** — multiple engineers and CI/CD pipelines need to read and write the same state
2. **State locking** — prevents two simultaneous applies from corrupting state (S3 native locking with `use_lockfile = true` in Terraform 1.10+, previously required a DynamoDB table)
3. **Durability** — local state gets lost when you delete the directory or the CI runner is destroyed

*Project tie-in: zen-infra uses an S3 backend with `use_lockfile = true`. Each environment has its own state key: `envs/dev/terraform.tfstate`, `envs/qa/terraform.tfstate`, `envs/prod/terraform.tfstate` — complete isolation between environments.*

---

**Q18: What happens if two engineers run `terraform apply` at the same time?**

**A:** The first one acquires the state lock and proceeds. The second one tries to acquire the lock, finds it already held, and fails with:

```
Error: Error acquiring the state lock
Lock Info:
  ID:        abc-123
  Who:       engineer2@company.com
  Operation: apply
```

The second engineer must wait. If the first apply was interrupted (CI runner died mid-apply), the lock may be left stale. You can release it with `terraform force-unlock <lock-id>` — but only after confirming no apply is actually running, otherwise you risk state corruption.

*Project tie-in: The GitHub Actions workflow uses `concurrency: cancel-in-progress: false` to queue runs rather than let them race. The S3 locking is a second layer of protection.*

---

**Q19: What is `terraform import` and when would you use it?**

**A:** `terraform import` brings an existing real AWS resource under Terraform management by writing it into state — without creating or destroying it.

Use cases:
- Resources were created manually before Terraform was adopted
- Terraform state was lost but the infrastructure still exists
- Migrating resources from another Terraform workspace

Modern approach (Terraform 1.5+) uses declarative import blocks in `.tf` files:

```hcl
import {
  to = module.ecr.aws_ecr_repository.main["api-gateway"]
  id = "api-gateway"
}
```

This is better for CI/CD — the import is codified, reviewable in a PR, and idempotent.

*Project tie-in: ECR repositories survived a `terraform destroy` because they contained images (AWS blocks deletion of non-empty repos). When the state was gone but repos still existed, import blocks were used to adopt them back into state without recreating them and losing the images.*

---

**Q20: A `terraform apply` failed halfway through. Half the resources were created, half were not. What do you do?**

**A:** Do not panic — Terraform is designed for this. The state file was updated for resources that successfully created. Steps:

1. **Read the error** — understand why it failed (permissions, AWS service limit, configuration error)
2. **Fix the root cause** — update Terraform code or AWS permissions
3. **Run `terraform plan`** — Terraform shows only the remaining resources to create. Already-created ones are in state and will not be recreated.
4. **Run `terraform apply`** — it picks up where it left off

The dangerous scenario is a resource that partially created (e.g., EKS cluster stuck in `CREATING` state when the runner died). Run `terraform refresh` to sync state with actual AWS resource states, then re-apply.

Never run `terraform state rm` as a first response — understand the situation fully before removing state entries.

---

## 3.2 Modules

### Concept

A Terraform module is a reusable group of resources with defined inputs (variables) and outputs. Think of it like a function — takes inputs, creates resources, returns outputs.

When to create a module:
- Same resource group used in multiple environments with different parameters
- Resources that always go together (VPC + subnets + route tables + IGW)
- You want to hide implementation complexity from the caller

When NOT to create a module:
- A single resource used once
- Premature abstraction before you have two real use cases

### Practice Activity

```bash
# 1. Read through the module structure
cat zen-infra/modules/vpc/main.tf
cat zen-infra/modules/vpc/variables.tf
cat zen-infra/modules/vpc/outputs.tf

# 2. Trace how envs/dev/main.tf calls the vpc module and passes variables
# 3. Find where module outputs are consumed:
#    - module.vpc.private_eks_subnet_ids → used by module.eks
#    - module.vpc.private_rds_subnet_ids → used by module.rds
#    - module.vpc.vpc_id → used by module.rds
# 4. Add a new output to modules/ecr/outputs.tf:
#    output "repository_urls" {
#      value = { for k, v in aws_ecr_repository.main : k => v.repository_url }
#    }
# 5. Reference it in envs/dev/outputs.tf:
#    output "ecr_repository_urls" {
#      value = module.ecr.repository_urls
#    }
# 6. Run terraform plan — no infrastructure changes, just new outputs
```

---

### Interview Questions

---

**Q21: What is a Terraform module and how do you decide what to put in one?**

**A:** A module is a reusable group of resources with defined inputs and outputs — like a function. The caller provides inputs, the module creates resources, and exposes outputs that other modules can consume.

Decision criteria:
- **Reusability** — if the same resource group is used in dev, qa, and prod with different parameters, make it a module
- **Encapsulation** — if a set of resources always go together (VPC + subnets + route tables + IGW + NAT), group them
- **Complexity hiding** — if implementation details are noise for the caller, module them

Avoid over-modularizing. A module for a single `aws_s3_bucket` is overkill. A module for "a complete VPC with public and private subnets, NAT gateway, and routing" makes sense.

*Project tie-in: The project has modules for vpc, eks, rds, ecr, iam, and secrets-manager. Each environment calls the same modules with different parameters — node instance type, desired capacity, subnet CIDRs — without duplicating any resource definitions.*

---

**Q22: How do you manage sensitive values in Terraform? What should never be in a `.tfvars` file committed to git?**

**A:** Sensitive values (DB passwords, API keys, private keys) should never be in committed files. Options:

1. **Pass via CI/CD** — secrets stored in GitHub Secrets, passed as `-var` flags at plan and apply time: `-var="db_password=${{ secrets.DEV_DB_PASSWORD }}"`. The value never touches the filesystem.
2. **AWS Secrets Manager or SSM Parameter Store** — store the secret there, read it in Terraform with a `data` source. The value never appears in `.tf` files.
3. **HashiCorp Vault** — dynamic secrets with time-limited credentials.

What never goes in `.tfvars`: passwords, API keys, private keys, tokens. Even if the file is `.gitignored`, it will eventually be committed by accident. Design the workflow so secrets never need to be on disk at all.

*Project tie-in: `db_password` and `jwt_secret` are Terraform variables with no default value. They are passed via `-var` from GitHub Secrets at plan and apply time. There is no `terraform.tfvars` file — values are never stored on disk.*

---

## 3.3 Workspaces vs Directory-per-Environment

### Concept

Two approaches to managing multiple environments in Terraform:

**Workspaces** — single configuration, multiple state files. Switch with `terraform workspace select qa`.

**Directory-per-environment** — separate `envs/dev/`, `envs/qa/`, `envs/prod/` directories each with their own configuration and backend.

### Practice Activity

```bash
cd zen-infra/envs/dev

# 1. Check current workspace
terraform workspace list
# You will see: * default

# 2. Understand why this project uses directories instead of workspaces
# Compare envs/dev/main.tf and envs/qa/main.tf:
#   - Different instance sizes
#   - Different capacity settings
#   - Different subnet CIDRs
# This configuration difference is explicit in directories,
# but would require messy terraform.workspace conditionals with workspaces

# 3. Create a test workspace (just to see how they work)
terraform workspace new test
terraform workspace list
terraform workspace select default
terraform workspace delete test
```

---

### Interview Questions

---

**Q23: What is the difference between Terraform workspaces and separate environment directories? Which do you prefer?**

**A:**
- **Workspaces** — single configuration, multiple state files. Switch context with `terraform workspace select qa`. State stored as separate files per workspace in the backend.
- **Directory-per-environment** — separate `envs/dev/`, `envs/qa/`, `envs/prod/` directories, each with their own configuration, variables, and backend key.

I prefer directories for production infrastructure because:

1. **Blast radius** — a mistake in `envs/dev` cannot affect `envs/prod`. Different state, different providers, different working directory. With workspaces, one `terraform workspace select` mistake away from applying dev config to prod.
2. **Configuration drift** — dev and prod legitimately differ (node count, instance size, which modules are enabled). Directories make this explicit and clean. Workspaces rely on `terraform.workspace` conditionals that grow messy.
3. **Clarity** — `cd envs/prod && terraform plan` is unambiguous. `terraform workspace select prod && terraform plan` requires remembering to switch before every operation.

Workspaces are fine for ephemeral environments (feature branch environments, PR previews) but not for persistent dev/qa/prod.

---

**Q24: How do you handle Terraform provider version upgrades safely?**

**A:** Major version upgrades (e.g., AWS provider 5.x → 6.x) require a deliberate process:

1. **Read the changelog** — check for breaking changes, removed resources, renamed attributes
2. **Update version constraint in a feature branch** — change `~> 5.0` to `~> 6.0`
3. **Run `terraform init -upgrade`** — downloads the new provider version
4. **Run `terraform plan`** — look for unexpected changes. Breaking changes show as destroy and recreate rather than update-in-place
5. **Fix any breaking changes** in Terraform code before merging
6. **Test in dev first** — apply in dev, verify infrastructure works, then promote to qa and prod

Never let automated tools like Dependabot auto-merge major provider version upgrades. The PR should be reviewed by an engineer who has read the changelog.

*Project tie-in: Dependabot opened PRs to upgrade `hashicorp/aws` from `~> 5.0` to `~> 6.40` and `hashicorp/kubernetes` from `~> 2.0` to `~> 3.0`. These were closed intentionally — major version upgrades require deliberate review and testing, not automatic merging.*

---

## 3.4 Plan and Apply Pipeline

### Concept

The production-safe Terraform workflow:

```
Feature branch
    │
    ▼
PR → plan runs automatically → plan posted as PR comment
    │
    ▼
Merge to main → plan runs again → upload tfplan artifact
    │
    ▼
Apply job pauses → "Review deployments" approval gate
    │
    ▼
Engineer reviews plan comment → approves → apply runs using saved tfplan
```

The key principle: **apply executes exactly what was reviewed, not a new plan**.

### Practice Activity

```bash
# 1. Make a real Terraform change — change desired_capacity in envs/dev/main.tf
#    from 3 to 2

# 2. Create a feature branch and push
git checkout -b feature/test-plan-apply
git add envs/dev/main.tf
git commit -m "test: reduce desired capacity to 2"
git push origin feature/test-plan-apply

# 3. Open a PR — observe the plan job runs automatically
# 4. Read the plan comment on the PR:
#    - Should show the EKS node group change
#    - Format/Init/Validate/Plan status table
# 5. Merge the PR
# 6. Observe: plan runs again on main, apply job appears waiting for approval
# 7. Click "Review deployments" → review the plan → Approve and deploy
# 8. Observe apply runs using the downloaded tfplan artifact
# 9. Revert the change after testing
```

---

### Interview Questions

---

**Q25: What is the difference between `terraform plan -out=tfplan` and running `terraform apply` directly?**

**A:** `terraform plan -out=tfplan` saves the exact plan to a binary file. `terraform apply tfplan` then executes exactly that saved plan with no re-evaluation.

`terraform apply` without a plan file runs a fresh plan first and applies it immediately. This is convenient locally but risky in CI/CD because infrastructure state might change between when you reviewed the plan and when apply runs.

The `plan -out` and `apply tfplan` pattern ensures **what you approved is exactly what gets applied**. This is the standard for any production pipeline.

*Project tie-in: The plan job saves `-out=tfplan`, uploads it as a GitHub artifact with 1-day retention. The apply job downloads it and runs `terraform apply tfplan`. The approval gate sits between — you approve the exact plan that was generated, not a re-generated one.*

---

**Q26: Why would you run `terraform plan` again on main branch after it already ran on the PR?**

**A:** Because the infrastructure state may have changed between when the PR was opened and when it was merged. Scenarios:

- Another PR was merged and applied while yours was open
- Someone manually changed something in AWS
- A scheduled drift-correction job ran

If you apply the plan that was generated 2 days ago when the PR was opened, you might be applying against a different base state than expected, causing unexpected changes or failures.

Running a fresh plan on main immediately before apply ensures the plan reflects the actual current state of infrastructure at the moment of apply — not a stale snapshot from the PR.

---

---

# QUICK REFERENCE

## Question Difficulty by Level

| Level | Topics |
|---|---|
| **Junior (0–2 yrs)** | What is a branch, what is a PR, what is terraform init/plan/apply, what are secrets |
| **Mid (2–4 yrs)** | Branching strategies, CI/CD pipeline structure, Terraform modules, state backends, environments |
| **Senior (5–6 yrs)** | Concurrency and state locking, plan/apply separation with approval gates, reusable workflows, import blocks, provider upgrades, workspace vs directory, debugging partial applies |
| **Staff/Principal** | Multi-region strategies, Terraform at scale (Terragrunt), drift detection automation, policy-as-code (OPA/Sentinel), secret rotation strategies |

---

## Answer Framework for Senior Level

Every answer should follow this structure:

```
1. Define the concept clearly (1–2 sentences)
2. Explain WHY it matters / what problem it solves
3. Describe HOW you used it in a real project
4. Mention the trade-off or alternative you considered
5. State the outcome
```

Interviewers at this level are not testing whether you know what Terraform is. They are testing whether you have made real decisions under real constraints and learned from them.

---

## Common Red Flags Interviewers Watch For

| What you say | What they hear |
|---|---|
| "I always use `terraform apply` directly" | Has not worked in a team or production environment |
| "We store secrets in `.tfvars`" | Security gap — will introduce credential leaks |
| "I cancel the apply if it takes too long" | Does not understand partial state risk |
| "We push directly to main" | No peer review, no change control |
| "Dependabot handles our upgrades" | Not reviewing major version breaking changes |
| "We use one big Terraform file" | Has not worked on a project at scale |

---

*This guide is built around the zen-pharma infrastructure project. All examples and project tie-ins reference real decisions made in that codebase.*
