# Deployment Control Repo

This repository is a lightweight deployment trigger/control layer.

It does **not** contain the main CI/CD deployment code, server SSH credentials,
Docker deployment scripts, or application source code.

Instead, this repo exposes GitHub Actions workflows that allow an authorized
user/client to request deployments. Each workflow then triggers the actual
deployment workflow in the private main CI/CD repository, with
`deployer: Goyal-co` so runs are auditable on the CI/CD side.

---

## What this repo does

This repo provides a controlled GitHub Actions UI for deployments.

Flow:

```text
User runs a Deploy * workflow in this control repo
        |
        v
This repo calls GitHub Actions API
        |
        v
Private main CI/CD repo workflow is triggered
  (2. Deploy Services · <project> · Goyal-co)
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
deployer          ← always Goyal-co from these callers
project
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

## Workflows in this repo

Copy the matching file from `CI-CD-template` (repo root examples) into
`.github/workflows/` here:

| Workflow (Actions UI) | File | Fixed project | Fixed services |
| --- | --- | --- | --- |
| Deploy Booking Inventory | `deploy-booking-inventory.yml` | `booking-inventory` | all with `build:` |
| Deploy Construct IQ | `deploy-construct-iq.yml` | `ciq` | all with `build:` |
| Deploy PartnerGoyal | `deploy-partnergoyal.yml` | `partnergoyal` | `app` |
| Deploy TitanCRM | `deploy-titancrm.yml` | `titancrm` | all with `build:` |

Each caller is **fixed-config** (`workflow_dispatch` with no extra inputs). Defaults:

```text
environment:     goyal-main-ec2
deployer:        Goyal-co
build:           true
action:          rollout
force_recreate:  false
sudo:            true
pull_env:        false
```

To change behavior, edit the `env:` block in that workflow file (not GitHub secrets).

---

## Required Actions configuration

```text
Settings -> Secrets and variables -> Actions
```

### Repository secret (1)

#### `MAIN_REPO_ACTIONS_TOKEN`

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

### Repository variables (4)

These are **not** secrets — they are plain config. Set them under:

```text
Settings -> Secrets and variables -> Actions -> Variables
```

| Variable | Default value | Description |
| --- | --- | --- |
| `TARGET_OWNER` | `dhimanparas20` | Owner of the private CI/CD repo |
| `TARGET_REPO` | `CI-CD-template` | Private CI/CD repo name |
| `TARGET_WORKFLOW` | `2_deploy-services.yml` | Workflow **file name** under `.github/workflows/` |
| `TARGET_REF` | `main` | Branch/ref that contains that workflow |

#### `TARGET_WORKFLOW` — file name, not display name

The main CI/CD workflow starts with:

```yaml
name: 2. Deploy Services
```

That does **not** mean `TARGET_WORKFLOW` should be `2. Deploy Services`.

Use the actual file name:

```text
2_deploy-services.yml
```

(underscore after `2`, as in the CI/CD-template repo — not `2-deploy-services.yml`)

---

## Config summary

| Name | Type | Default / example |
| --- | --- | --- |
| `MAIN_REPO_ACTIONS_TOKEN` | **Secret** | `github_pat_xxx` |
| `TARGET_OWNER` | **Variable** | `dhimanparas20` |
| `TARGET_REPO` | **Variable** | `CI-CD-template` |
| `TARGET_WORKFLOW` | **Variable** | `2_deploy-services.yml` |
| `TARGET_REF` | **Variable** | `main` |

---

## Secrets that should NOT be added here

The control repo should not contain server or deployment secrets.

Do not add these to this repo:

```text
SSH_HOST
SSH_USER
SSH_PASSWORD
SSH_PRIVATE_KEY
CORE_DIR
WORK_DIR
CICD_REPO
GH_PAT
GIST_MAP
GIST_ID
GIST_FILES
```

These belong only in the private main CI/CD repo (GitHub Environment /
repository secrets there).

---

## What each fixed input means (main CI/CD)

Sent to **2. Deploy Services** on `dhimanparas20/CI-CD-template`:

### `environment`

```text
goyal-main-ec2
```

### `deployer`

```text
Goyal-co
```

Shows up in the CI/CD Actions list run title and deployment summary so you can
tell control-repo triggers apart from runs started as `dhimanparas20`.

### `project` / `services`

Set per workflow (see table above). Empty `services` means all services with a
`build:` in that project’s compose file.

### `build`

```text
true   ← default in these callers
false
```

### `action`

```text
restart | rollout | up
```

Default in these callers: `rollout` (`docker rollout`).

### `force_recreate`

Only meaningful when `action=up`. Default: `false`.

### `sudo`

Default: `true` (run docker with sudo on the server).

### `pull_env`

Default: `false`. If `true`, the **main** CI/CD repo must have gist-related
secrets (`GH_PAT`, `GIST_MAP`, etc.) — not this control repo.

---

## How to run a deployment

1. Open this control repository on GitHub.
2. Go to **Actions**.
3. Select one of:

```text
Deploy Booking Inventory
Deploy Construct IQ
Deploy PartnerGoyal
Deploy TitanCRM
```

4. Click **Run workflow** → **Run workflow**.

This dispatches **2. Deploy Services** in the private main CI/CD repo.

Check the real deploy logs there (not only in this control repo).

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

Recommended expiration: `30` or `90` days.

When the token expires:

1. Create a new fine-grained PAT.
2. Scope it only to `dhimanparas20/CI-CD-template`.
3. Permissions: Actions (R/W), Contents (R), Metadata (R).
4. Replace repository secret `MAIN_REPO_ACTIONS_TOKEN`.

---

## Troubleshooting

### Error: `Missing secret: MAIN_REPO_ACTIONS_TOKEN`

Add the repository **secret** `MAIN_REPO_ACTIONS_TOKEN`.

### Error: `Missing variable: TARGET_OWNER` (or other TARGET_*)

Add the repository **variables** (not secrets):

```text
TARGET_OWNER
TARGET_REPO
TARGET_WORKFLOW
TARGET_REF
```

```text
Settings -> Secrets and variables -> Actions -> Variables
```

### Error: `Resource not accessible by personal access token`

The token cannot dispatch on the main repo. Fix PAT repo access + Actions write,
then replace `MAIN_REPO_ACTIONS_TOKEN`.

### Error: `Not Found`

Usually wrong:

```text
TARGET_OWNER / TARGET_REPO / TARGET_WORKFLOW / TARGET_REF
```

Confirm `TARGET_WORKFLOW` is exactly:

```text
2_deploy-services.yml
```

### Workflow triggers successfully but deployment does not run

Open **Actions** on `dhimanparas20/CI-CD-template`. Look for a run titled like:

```text
2. Deploy Services · <project> · Goyal-co
```

The control repo only sends the request; deploy logs live in the main CI/CD repo.

---

## Architecture summary

```text
Control repo
  Secret:
    MAIN_REPO_ACTIONS_TOKEN

  Variables (defaults):
    TARGET_OWNER=dhimanparas20
    TARGET_REPO=CI-CD-template
    TARGET_WORKFLOW=2_deploy-services.yml
    TARGET_REF=main

  Workflows:
    Deploy Booking Inventory
    Deploy Construct IQ
    Deploy PartnerGoyal
    Deploy TitanCRM
      deployer=Goyal-co (fixed)

        |
        | GitHub Actions workflow_dispatch API
        v

Private main CI/CD repo (dhimanparas20/CI-CD-template)
  Workflow:
    2. Deploy Services   (.github/workflows/2_deploy-services.yml)

  Environment secrets/vars (e.g. goyal-main-ec2):
    SSH_HOST, SSH_USER, SSH_PRIVATE_KEY / SSH_PASSWORD
    CORE_DIR, CICD_REPO, …
    (optional GH_PAT / GIST_MAP if pull_env=true)

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

Current expected target workflow file:

```text
2_deploy-services.yml
```

Caller examples (canonical copies) also live at the root of CI/CD-template:

```text
deploy-booking-inventory.yml
deploy-construct-iq.yml
deploy-partnergoyal.yml
deploy-titancrm.yml
```

Copy those into this control repo’s `.github/workflows/` and use this document
as the control-repo `README.md`.
