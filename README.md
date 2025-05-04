# 🛠️ GitHub Workflows – Shared CI/CD

This repository contains **centralised CI/CD workflows** for use across multiple project repos. It enables:

- ✅ Docker-based **CI pipelines** with optional Trivy & Copa security scanning
- ✅ AWS SAM-based **CD pipelines** for deploying serverless apps
- ✅ Clean secret isolation — secrets stay in this repo
- ✅ Reusable and dispatch-based architecture
- ✅ Project workflows stay simple and declarative


## 🔐 Why this design?
GitHub **does not allow secrets in a reusable workflow to be accessed** when it is called from a different repo. To solve this:

- Project repos use lightweight trigger workflows

- These triggers call the execution workflows in the same repo using the GitHub API

- Execution workflows run in the shared repo, so they have access to secrets like AWS and Docker credentials

This structure enables secure, scalable, and centralised CI/CD across multiple independent services with minimal setup in each project repo.


---

## 🔃 Workflow Architecture

```
project-repo
├── .github/workflows/build.yml       ← triggers CI workflow
├── .github/workflows/deploy.yml      ← triggers CD workflow
↓
github-workflows (this repo)
├── trigger-build-and-push-image.yaml → dispatches CI
├── exec-build-and-push-image.yaml    → builds Docker image
├── trigger-deploy-aws-sam.yaml       → dispatches CD
└── exec-deploy-aws-sam.yaml          → deploys SAM stack
```

---

## 🧱 Workflows in This Repo

### 🔧 CI Workflows

| Purpose          | Workflow File                       | Location     |
| ---------------- | ----------------------------------- | ------------ |
| Project trigger  | `example-project-repo-ci.yaml`                         | Project repo |
| Dispatch trigger | `trigger-build-and-push-image.yaml` | Shared repo  |
| Execution        | `exec-build-and-push-image.yaml`    | Shared repo  |


### CD Workflows

| Purpose          | Workflow File                 | Location     |
| ---------------- | ----------------------------- | ------------ |
| Project trigger  | `example-project-repo-aws-sam-cd.yaml`                  | Project repo |
| Dispatch trigger | `trigger-deploy-aws-sam.yaml` | Shared repo  |
| Execution        | `exec-deploy-aws-sam.yaml`    | Shared repo  |


### 🚀 `exec-build-and-push-image.yaml`

Builds a Docker image, optionally scans it with Trivy, patches it with Copa, and pushes it to Docker Hub.

**Inputs:**
- `dockerfile-path` (string, required) – path to Dockerfile in project repo
- `image-name` (string, required) – Docker image name
- `image-tag` (string, default: `latest`)
- `run-trivy-scan` (boolean, default: `false`)
- `run-copa-patch` (boolean, default: `false`)

---

### ⚡ `trigger-build-and-push-image.yaml`

Reusable workflow that receives input from a project repo and uses the GitHub API to trigger `exec-build-and-push-image.yaml`.

This (and all other) intermediate trigger workflow exists to **ensure secrets remain isolated within the shared workflows repository**.

GitHub Actions **does not allow secrets from the reusable workflow’s repository to be accessed** when the reusable workflow is called from another repository.

By dispatching the actual execution workflow from within the same repo (via the GitHub API), this design ensures the job runs with access to the **correct secret scope**, while keeping project repositories free of sensitive credentials.

**Secrets required in this repo:**
- `GH_PAT_TOKEN` – a personal access token with `repo` and `workflow` scope

---

### 🚢 `exec-deploy-aws-sam.yaml`

Deploys an AWS SAM app using pre-built Docker image or CodeUri. Supports parameter overrides.

**Inputs:**
- `stack-name` (string)
- `s3-bucket` (string)
- `region` (string)
- `template` (string, default: `template.yaml`)
- `parameters` (string) – e.g. `--parameter-overrides Key1=Value1 Key2=Value2`

**Secrets required in this repo:**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

---

### ⚡ `trigger-deploy-aws-sam.yaml`

Reusable workflow that triggers `exec-deploy-aws-sam.yaml` via GitHub API with the appropriate parameters.

**Secrets required in this repo:**
- `GH_PAT_TOKEN` – a personal access token with `repo` and `workflow` scope

---

## 🧪 Example Project Workflows

### 🧱 `build.yml` – CI from project repo

```yaml
name: Trigger Shared Docker CI

on:
  push:
    branches: [main]

jobs:
  trigger-build:
    uses: blee1077/github-workflows/.github/workflows/trigger-build-and-push-image.yaml@main
    with:
      dockerfile-path: ./code_dir
      image-name: image-name
      image-tag: latest
      run-trivy-scan-and-copa-patch: true
    secrets: inherit
```

---

### ☁️ `deploy.yml` – CD from project repo

```yaml
name: Deploy via Shared SAM CD

on:
  push:
    branches: [main]

  workflow_dispatch:
    inputs:
      stack-name:
        required: false
        type: string
        default: sam-app
      parameters:
        required: false
        type: string
        default: |
          --parameter-overrides \
          Param1=key1 \
          Param2=key2

jobs:
  trigger-shared-deploy:
    uses: blee1077/github-workflows/.github/workflows/trigger-deploy-aws-sam.yaml@main
    with:
      stack-name: ${{ github.event_name == 'workflow_dispatch' && inputs.stack-name || 'sam-app' }}
      s3-bucket: your-sam-s3-bucket-for-source-control
      region: eu-west-2
      parameters: ${{ github.event_name == 'workflow_dispatch' && inputs.parameters || '...' }}
    secrets: inherit
```

---

## 🛡️ Secrets Handling

Secrets like Docker credentials and AWS keys should be stored in **this shared repo** only:

| Secret Name             | Used For                      |
|-------------------------|-------------------------------|
| `DOCKERHUB_USERNAME`    | Docker image login            |
| `DOCKERHUB_TOKEN`       | Docker image login            |
| `AWS_ACCESS_KEY_ID`     | CD workflow (SAM deploy)      |
| `AWS_SECRET_ACCESS_KEY` | CD workflow (SAM deploy)      |

> 🔒 Project repositories **do not store secrets**. Secrets are managed entirely within the shared workflows repository. When a workflow is triggered, the **execution workflow inherits secrets from the trigger workflow in the same repo**. The only secret required in the project repo is `GH_PAT_TOKEN`, used to authorise the workflow trigger.

---

## 🌱 Creating a New Project

To onboard a new repo with CI/CD:

1. Copy the example `example-project-repo-ci.yaml` and `example-project-repo-aws-sam-cd.yml` into the project
2. Set:
   - Dockerfile path
   - Image name/tag
   - Stack name
   - Any SAM parameter overrides
3. Ensure the shared repo has required secrets
4. Push to `main` or run manually via the GitHub UI
