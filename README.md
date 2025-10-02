# GITHUB ACTIONS AND CI/CD COURSE: DEPLOYMENT PIPELINES AND CLOUD PLATFORMS
INTRODUCTION TO GITHUB ACTIONS COURSE: DEPLOYMENT AND CLOUD INTEGRATION
Welcome to our comprehensive course on GitHub Actions, focusing on deployment pipelines and cloud platform integration. In this course, we’re going to explore how you can leverage GitHub Actions to automate your deployment process, effectively pushing your applications to various cloud environments. Whether you’re a building developer, a seasoned engineer, or anyone interested in the world of DevOps, this course is tailored to provide you practical skills and insights into the world of continuous integration, continuous deployment, and cloud services.

---

## Detailed step-by-step for instructors & students (practical checklist)

### 0. Prep (before first practical)

1. Create a starter repository with:

   * `README.md`
   * sample Node.js app (or minimal app for chosen stack)
   * test script (e.g., `npm test`)
   * `.github/workflows/` directory
2. In GitHub repo → Settings → Secrets and variables → Actions: add required secrets (names below).
3. Ensure cloud account IAM user/service principal with minimal deploy permissions and keys stored in repo secrets.

---

### 1. Basic CI workflow (lint → test → build)

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

Deliverable: passing pipeline on every PR.

---

### 2. Automated version bump & tag on main

Use a tagger action (example uses `anothrNick/github-tag-action`, but make sure to pin versions in production).

`.github/workflows/bump-and-tag.yml`:

```yaml
name: Bump version and tag

on:
  push:
    branches:
      - main

jobs:
  bump:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Bump version and push tag
        uses: anothrNick/github-tag-action@v1.26.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DEFAULT_BUMP: patch
```

Notes:

* `fetch-depth: 0` is required so the action sees history and tags.
* Use `DEFAULT_BUMP` = `major|minor|patch` (you can add commit convention parsing for automatic bump type).

---

### 3. Create GitHub Release when tag is pushed

`.github/workflows/release.yml`:

```yaml
name: Create Release

on:
  push:
    tags:
      - '*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Create release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref_name }}
          release_name: Release ${{ github.ref_name }}
          draft: false
          prerelease: false
```

Explanation:

* `github.ref_name` extracts the tag string (e.g., `v1.2.3`) cleanly.
* You can upload build artifacts using `actions/upload-release-asset@v1` after building.

---

### 4. Deploy to AWS (example)

Secrets required in repo: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`. (Store them in GitHub Secrets.)

`.github/workflows/deploy-aws.yml`:

```yaml
name: Deploy to AWS

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Build artifact
        run: |
          npm ci
          npm run build

      - name: Deploy (example using S3 + CloudFront)
        run: |
          aws s3 sync ./build s3://your-bucket-name --delete
          aws cloudfront create-invalidation --distribution-id YOUR_DIST_ID --paths "/*"
```

Notes:

* For containerized apps, build/push to ECR and update ECS or EKS manifests.
* For complex infra, use IaC (CloudFormation / Terraform) triggered from Actions.

---

### 5. Deploy to Azure (example)

Secrets: `AZURE_CREDENTIALS` (service principal JSON), `AZURE_RESOURCE_GROUP`, `AZURE_WEBAPP_NAME`.

Basic deploy snippet:

```yaml
- name: Login to Azure
  uses: azure/login@v1
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}

- name: Deploy to Azure Web App
  uses: azure/webapps-deploy@v2
  with:
    app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
    package: ./build
```

Instructor note: show students how to create a service principal and generate the JSON for `AZURE_CREDENTIALS`.

---

### 6. Deploy to GCP (example)

Secrets: `GCP_SA_KEY` (service account JSON), `GCP_PROJECT`, `GKE_CLUSTER`, etc.

Login & deploy snippet:

```yaml
- name: Set up GCP credentials
  uses: google-github-actions/setup-gcloud@v1
  with:
    service_account_key: ${{ secrets.GCP_SA_KEY }}
    project_id: ${{ secrets.GCP_PROJECT }}

- name: Deploy to GKE
  run: |
    gcloud container clusters get-credentials $GKE_CLUSTER --zone $GKE_ZONE
    kubectl apply -f k8s/deployment.yaml
```

---

### 7. Environments, approvals, and protection

1. Create GitHub Environments: `staging`, `production`.
2. Add required reviewers for `production` environment (protect deployment).
3. Use `environment: production` in workflow jobs to trigger required reviewers and secrets scoped to that environment:

```yaml
deploy-prod:
  runs-on: ubuntu-latest
  environment: production
  steps:
    ...
```

---

### 8. Canary / Blue-Green and rollback (pattern)

* Canary: deploy version `vX` to a small subset of instances; run automated health checks; promote or rollback.
* Blue-Green: keep blue (current) and green (new) environments; switch a load balancer after smoke tests pass.

Practical lab: implement a canary script that updates deployment weights (or routes), waits `n` seconds, runs health checks, and either shifts 100% or reverts.

---

## Assessment & assignments (grading checklist)

* **Lab checks (pass/fail):**

  * CI builds & tests on PRs — pass
  * Auto version bump & tag — pass
  * Release created from tag — pass
  * Deployment to staging — pass
  * Protected production deploy with approval — pass
* **Capstone rubric:**

  * pipeline correctness (40%)
  * security & secrets handling (20%)
  * deployment strategy & rollback (20%)
  * documentation & README (20%)

Deliverable for capstone: a public repo (or private + instructor access) with workflows in `.github/workflows/`, clear README with diagrams, and demo screenshots or short recording.

---

## Teaching materials & student aids

1. Quick reference cards:

   * GitHub Actions keywords: `on`, `jobs`, `steps`, `uses`, `run`, `needs`, `env`, `secrets`, `with`.
   * Events cheat sheet: `push`, `pull_request`, `workflow_dispatch`, `release`, `schedule`.
2. Troubleshooting checklist:

   * Check action logs
   * Confirm secrets exist and have correct names
   * Ensure correct token scope / `permissions` in workflow if accessing the API
   * Validate YAML syntax (yamlint.com)
3. Starter project templates:

   * Node.js web app (build → dist)
   * Simple Docker-based service with `Dockerfile`
4. Diagrams: pipeline flow, branching model, environment promotion flow (staging → canary → production).

---

## Useful policy & best practices (class discussion)

* Least privilege credentials — give Actions only necessary permissions.
* Rotate and scope secrets; prefer environment-scoped secrets for production.
* Pin actions to major versions, and for high-security orgs pin full SHA.
* Monitor Actions usage to control cost and detect unusual workflows.

---

## Example assignments (practical)

1. Short assignment: Add CI that fails if coverage < 80% and prints test report artifacts.
2. Medium: Add automated semantic release using commit message conventions (Conventional Commits) so `major|minor|patch` is inferred.
3. Long (capstone): Full pipeline — build, containerize, push to registry, deploy to cloud, produce release notes automatically, implement canary with health checks and rollback.

---

## Extra: common YAML gotchas & fixes

* Use `$GITHUB_ENV` to persist environment variables between steps.
* Use `fetch-depth: 0` when actions require tag history.
* Correct secrets syntax: `secrets.MY_SECRET` (no extra quotes). Example: `aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}`.

---
