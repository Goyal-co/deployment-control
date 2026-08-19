# Deployment Control Repo

This repository is a lightweight deployment trigger/control layer.

It does **not** contain the main CI/CD deployment code, server SSH credentials,
Docker deployment scripts, or application source code.

Instead, this repo exposes a GitHub Actions workflow that allows an authorized
user/client to request deployments. The workflow then triggers the actual
deployment workflow in the private main CI/CD repository.

---

## What this repo does

This repo provides a controlled GitHub Actions UI for deployments.

Flow:

```text
User runs workflow in this control repo
        |
        v
This repo calls GitHub Actions API
        |
        v
Private main CI/CD repo workflow is triggered
        |
        v
Main CI/CD repo connects to server via SSH
        |
        v
Docker Compose services are deployed/restarted/rolled out
```

The actual deployment happens from the private CI/CD repository.

This repo only sends deployment inputs such as:

```text
environment
services
build
action
force_recreate
sudo
pull_env
```

---

## Main use case

This repository is useful when you want to allow someone to trigger deployments
without giving them access to the private CI/CD repository or its source code.

The user/client can run deployments from this control repo, while the real
deployment logic and secrets stay protected inside the main CI/CD repo.

---

## Required GitHub Actions secrets

Add these in this repository:

```text
Settings -> Secrets and variables -> Actions -> Repository secrets
```

### `MAIN_REPO_ACTIONS_TOKEN`

GitHub fine-grained personal access token used to trigger the workflow in the
private main CI/CD repo.

Example value:

```text
github_pat_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Required permissions for this token:

```text
Repository access:
  Only selected repositories

Selected repository:
  dhimanparas20/CI-CD-template

Repository permissions:
  Actions: Read and write
  Contents: Read-only
  Metadata: Read-only
```

No access is required for:

```text
Deployments
Workflows
Issues
Pull requests
Administration
Secrets
```

---

### `TARGET_OWNER`

GitHub username or organization that owns the private main CI/CD repo.

Example value:

```text
dhimanparas20
```

---

### `TARGET_REPO`

Name of the private main CI/CD repository.

Example value:

```text
CI-CD-template
```

---

### `TARGET_WORKFLOW`

Actual workflow file name inside the main CI/CD repo.

This must be the file name inside:

```text
.github/workflows/
```

Example values:

```text
2-deploy-services.yml
```

or:

```text
deploy-services.yml
```

Important:

Do not use the workflow display name.

For example, if the main workflow starts with:

```yaml
name: 2. Deploy Services
```

that does not mean `TARGET_WORKFLOW` should be `2. Deploy Services`.

Use the actual file name, for example:

```text
2-deploy-services.yml
```

---

### `TARGET_REF`

Branch or Git ref where the target workflow exists in the main CI/CD repo.

Example value:

```text
main
```

If your main CI/CD workflow is on another branch, use that branch name.

Example:

```text
production
```

---

## Required secrets summary

| Secret name | Description | Example value |
| --- | --- | --- |
| `MAIN_REPO_ACTIONS_TOKEN` | Fine-grained GitHub token used to trigger the private CI/CD repo workflow | `github_pat_xxx` |
| `TARGET_OWNER` | Owner of the private CI/CD repo | `dhimanparas20` |
| `TARGET_REPO` | Private CI/CD repo name | `CI-CD-template` |
| `TARGET_WORKFLOW` | Workflow file name in `.github/workflows/` | `2-deploy-services.yml` |
| `TARGET_REF` | Branch/ref of the target workflow | `main` |

---

## Secrets that should NOT be added here

The control repo should not contain server or deployment secrets.

Do not add these secrets to this repo:

```text
SSH_HOST
SSH_USER
SSH_PASSWORD
SSH_PRIVATE_KEY
WORK_DIR
CICD_REPO
GH_PAT
GIST_ID
GIST_FILES
```

These belong only in the private main CI/CD repo.

---

## Deployment inputs

When running the workflow manually, GitHub will ask for these inputs.

### `environment`

Target deployment environment/server.

Current supported value:

```text
goyal-main-ec2
```

Example:

```text
goyal-main-ec2
```

---

### `services`

Comma-separated Docker Compose service names.

Example:

```text
app
```

Multiple services:

```text
app,worker,api
```

If left empty, the main CI/CD workflow may deploy all app services that have a
Dockerfile/build configuration.

Default value:

```text
app
```

---

### `build`

Whether to build Docker images before running the deployment action.

Example values:

```text
true
false
```

Recommended for most deploys:

```text
true
```

---

### `action`

Deployment action to perform after optional build.

Available values:

```text
restart
rollout
up
```

#### `restart`

Runs Docker Compose restart for selected services.

Useful when:

```text
Code/config already exists on server and only container restart is needed.
```

#### `rollout`

Runs `docker rollout` for selected services.

Useful when:

```text
You use docker-rollout for zero/minimal downtime deployments.
```

#### `up`

Runs:

```text
docker compose up -d
```

Useful when:

```text
You want Docker Compose to recreate/start containers based on latest compose config.
```

---

### `force_recreate`

Only used when:

```text
action=up
```

If true, the main workflow passes:

```text
--force-recreate
```

Example values:

```text
true
false
```

Recommended default:

```text
false
```

---

### `sudo`

Whether to run Docker/Docker Compose commands with sudo on the server.

Example values:

```text
true
false
```

Recommended default:

```text
true
```

---

### `pull_env`

Whether to pull latest environment files from Gist before deployment.

Example values:

```text
true
false
```

Recommended default:

```text
false
```

If set to `true`, the main CI/CD repo must have these secrets configured:

```text
GH_PAT
GIST_ID
GIST_FILES
```

Those secrets must exist in the main CI/CD repo, not this control repo.

---

## Example deployment runs

### Deploy app with build and rollout

```text
environment: goyal-main-ec2
services: app
build: true
action: rollout
force_recreate: false
sudo: true
pull_env: false
```

---

### Restart app without building

```text
environment: goyal-main-ec2
services: app
build: false
action: restart
force_recreate: false
sudo: true
pull_env: false
```

---

### Run docker compose up with force recreate

```text
environment: goyal-main-ec2
services: app
build: true
action: up
force_recreate: true
sudo: true
pull_env: false
```

---

### Pull latest env files before deployment

```text
environment: goyal-main-ec2
services: app
build: true
action: rollout
force_recreate: false
sudo: true
pull_env: true
```

---

## How to run deployment

1. Open this repository on GitHub.
2. Go to:

```text
Actions
```

3. Select:

```text
Deploy Services
```

4. Click:

```text
Run workflow
```

5. Fill deployment inputs.
6. Click:

```text
Run workflow
```

This will trigger the actual deployment workflow in the private main CI/CD repo.

---

## Security notes

This repo does not store server SSH secrets.

However, it does store a GitHub token that can trigger deployments in the main
CI/CD repo.

Anyone who can modify workflows in this repository may be able to misuse that
token.

Recommended access model:

```text
Give deployment users only the minimum access needed to run workflows.
Avoid giving unnecessary admin/write access to this repository.
```

For production-grade security:

```text
Use short-lived GitHub App tokens instead of personal access tokens.
Rotate MAIN_REPO_ACTIONS_TOKEN regularly.
Keep the token scoped only to the main CI/CD repo.
Use GitHub environments/approvals for production deployments.
```

---

## Token rotation

GitHub strongly recommends setting an expiration on personal access tokens.

Recommended expiration:

```text
30 days
```

or:

```text
90 days
```

When the token expires:

1. Create a new fine-grained PAT.
2. Give it access only to the main CI/CD repo.
3. Grant permissions:

```text
Actions: Read and write
Contents: Read-only
Metadata: Read-only
```

4. Replace this repository secret:

```text
MAIN_REPO_ACTIONS_TOKEN
```

---

## Troubleshooting

### Error: `Missing secret: TARGET_OWNER`

One of the required repository secrets is missing.

Check:

```text
Settings -> Secrets and variables -> Actions -> Repository secrets
```

Required secrets:

```text
MAIN_REPO_ACTIONS_TOKEN
TARGET_OWNER
TARGET_REPO
TARGET_WORKFLOW
TARGET_REF
```

---

### Error: `Resource not accessible by personal access token`

The token does not have enough access to trigger the main workflow.

Check the fine-grained PAT:

```text
Repository access:
  dhimanparas20/CI-CD-template

Permissions:
  Actions: Read and write
  Contents: Read-only
  Metadata: Read-only
```

After updating the token, replace the secret:

```text
MAIN_REPO_ACTIONS_TOKEN
```

---

### Error: `Not Found`

Usually one of these is wrong:

```text
TARGET_OWNER
TARGET_REPO
TARGET_WORKFLOW
TARGET_REF
```

Check that `TARGET_WORKFLOW` is the actual file name inside:

```text
.github/workflows/
```

Example:

```text
2-deploy-services.yml
```

---

### Workflow triggers successfully but deployment does not run

Check the Actions tab in the private main CI/CD repo.

The control repo only sends the request. The actual deployment logs are in the
main CI/CD repository.

---

## Architecture summary

```text
Control repo
  Secrets:
    MAIN_REPO_ACTIONS_TOKEN
    TARGET_OWNER
    TARGET_REPO
    TARGET_WORKFLOW
    TARGET_REF

  Workflow:
    Deploy Services

        |
        | GitHub Actions workflow_dispatch API
        v

Private main CI/CD repo
  Workflow:
    2. Deploy Services

  Secrets:
    SSH_HOST
    SSH_USER
    SSH_PASSWORD
    SSH_PRIVATE_KEY
    WORK_DIR
    CICD_REPO
    GH_PAT
    GIST_ID
    GIST_FILES

        |
        | SSH
        v

Server
  Docker Compose deployment
```

---

## Maintainer notes

Current expected target repository:

```text
dhimanparas20/CI-CD-template
```

Current expected environment:

```text
goyal-main-ec2
```

Current expected workflow input set:

```text
environment
services
build
action
force_recreate
sudo
pull_env
```
