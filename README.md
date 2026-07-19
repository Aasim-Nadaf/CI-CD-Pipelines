# CI/CD Pipelines & GitHub Actions

A hands-on learning repository covering Continuous Integration and Continuous Delivery (CI/CD) concepts, GitHub Actions workflows, and real-world pipeline patterns.

---

## Table of Contents

- [What is CI/CD?](#what-is-cicd)
- [Core Concepts](#core-concepts)
- [GitHub Actions Overview](#github-actions-overview)
- [Workflow Syntax Breakdown](#workflow-syntax-breakdown)
- [Pipeline Stages](#pipeline-stages)
- [Common Workflow Patterns](#common-workflow-patterns)
- [Environment Variables & Secrets](#environment-variables--secrets)
- [Artifacts & Caching](#artifacts--caching)
- [Repository Structure](#repository-structure)
- [Resources](#resources)

---

## What is CI/CD?

**Continuous Integration (CI)** is the practice of automatically building and testing code every time a developer pushes a change. It catches bugs early before they reach production.

**Continuous Delivery (CD)** extends CI by automatically deploying tested code to staging or production environments, ensuring software can be released at any time reliably.

```
Developer pushes code
        │
        ▼
   CI Pipeline runs
  ┌─────────────────┐
  │  Build → Test   │   ← Continuous Integration
  └─────────────────┘
        │ (if passed)
        ▼
   CD Pipeline runs
  ┌──────────────────────────────────┐
  │  Deploy to Staging → Production  │   ← Continuous Delivery/Deployment
  └──────────────────────────────────┘
```

---

## Core Concepts

| Term | Description |
|---|---|
| **Pipeline** | A sequence of automated steps that code goes through from commit to deployment |
| **Job** | A set of steps that run on the same runner (virtual machine) |
| **Step** | A single task inside a job — either a shell command or a pre-built action |
| **Runner** | The server/VM that executes jobs (`ubuntu-latest`, `windows-latest`, `macos-latest`) |
| **Trigger / Event** | What starts a workflow — a push, a PR, a schedule, or manual dispatch |
| **Action** | A reusable, pre-packaged step from the GitHub Marketplace or your own repo |
| **Artifact** | Files produced by a job (build output, test reports) that can be passed between jobs |
| **Secret** | Encrypted environment variables for sensitive values like API keys and tokens |

---

## GitHub Actions Overview

GitHub Actions is GitHub's built-in CI/CD platform. Workflows are defined in YAML files stored under `.github/workflows/` in your repository.

```
your-repo/
└── .github/
    └── workflows/
        ├── ci.yml          ← runs on every push/PR
        ├── cd.yml          ← runs on merge to main
        └── scheduled.yml   ← runs on a cron schedule
```

Every workflow file has three main sections:

```yaml
name: <workflow name>

on: <trigger events>

jobs:
  <job-id>:
    runs-on: <runner>
    steps:
      - <steps>
```

---

## Workflow Syntax Breakdown

### Full annotated example

```yaml
name: CI Pipeline

# Triggers: run on push to main, or on any pull request
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest          # runner environment

    steps:
      # 1. Check out the repo code onto the runner
      - name: Checkout code
        uses: actions/checkout@v4

      # 2. Set up the runtime (Node.js in this case)
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      # 3. Install dependencies
      - name: Install dependencies
        run: npm ci

      # 4. Run tests
      - name: Run tests
        run: npm test

      # 5. Build the project
      - name: Build
        run: npm run build
```

### Trigger types

```yaml
on:
  push:                        # on every push
    branches: [main, develop]
    paths: ['src/**']          # only if files under src/ changed

  pull_request:                # on PR open/update
    branches: [main]

  schedule:
    - cron: '0 6 * * 1'       # every Monday at 6:00 AM UTC

  workflow_dispatch:           # allows manual trigger from GitHub UI
```

### Job dependencies (needs)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "building..."

  test:
    runs-on: ubuntu-latest
    needs: build               # waits for 'build' to succeed first
    steps:
      - run: echo "testing..."

  deploy:
    runs-on: ubuntu-latest
    needs: [build, test]       # waits for both
    steps:
      - run: echo "deploying..."
```

### Matrix builds (test across multiple versions)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

---

## Pipeline Stages

A typical production CI/CD pipeline follows these stages:

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Source  │──▶│  Build   │──▶│   Test   │──▶│  Stage   │──▶│  Deploy  │
│  (push)  │   │ compile  │   │ unit/int │   │ QA/UAT   │   │   prod   │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

| Stage | What happens |
|---|---|
| **Source** | Developer pushes code; pipeline is triggered |
| **Build** | Code is compiled, bundled, or containerized (Docker image) |
| **Test** | Unit tests, integration tests, linting, code coverage checks |
| **Stage** | Deploy to a non-production environment for QA or UAT |
| **Deploy** | Release to production; may include rollback on failure |

---

## Common Workflow Patterns

### Pattern 1 — Basic CI (lint + test on every PR)

```yaml
name: CI

on: [pull_request]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

### Pattern 2 — Build and push Docker image

```yaml
name: Docker Build & Push

on:
  push:
    branches: [main]

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: myuser/myapp:latest
```

### Pattern 3 — Deploy to a server via SSH

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            git pull origin main
            npm ci --production
            pm2 restart myapp
```

---

## Environment Variables & Secrets

### Environment variables

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      NODE_ENV: production         # job-level env var

    steps:
      - name: Print env
        run: echo $NODE_ENV

      - name: Step-level env
        run: echo $API_URL
        env:
          API_URL: https://api.example.com   # step-level env var
```

### Secrets

Secrets are added under **Settings → Secrets and variables → Actions** in your GitHub repo. They are never exposed in logs.

```yaml
steps:
  - name: Use a secret
    run: echo "Deploying with token"
    env:
      API_TOKEN: ${{ secrets.API_TOKEN }}
```

### Contexts

GitHub provides built-in context objects you can reference in workflows:

```yaml
${{ github.ref }}           # branch or tag that triggered the workflow
${{ github.sha }}           # full commit SHA
${{ github.actor }}         # username of whoever triggered it
${{ github.repository }}    # owner/repo-name
${{ runner.os }}            # Linux / Windows / macOS
${{ env.MY_VAR }}           # access env variables
${{ secrets.MY_SECRET }}    # access secrets
```

---

## Artifacts & Caching

### Upload and download artifacts (pass files between jobs)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      - run: echo "deploy from dist/"
```

### Caching dependencies (speeds up workflows)

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Cache node_modules
    uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-

  - run: npm ci
```

---

## Repository Structure

```
CI-CD-pipelines/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── cd.yml
│       └── docker.yml
├── examples/
│   ├── basic-ci/
│   ├── matrix-build/
│   ├── docker-pipeline/
│   └── deploy-ssh/
├── notes/
│   ├── concepts.md
│   └── cheatsheet.md
└── README.md
```

---

## Resources

| Resource | Link |
|---|---|
| GitHub Actions Docs | https://docs.github.com/en/actions |
| Workflow Syntax Reference | https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions |
| GitHub Actions Marketplace | https://github.com/marketplace?type=actions |
| act — run Actions locally | https://github.com/nektos/act |
| YAML syntax basics | https://yaml.org/spec/1.2.2/ |

---

> **Learning path tip:** Start by writing a basic CI workflow that runs `echo "hello"` on push, then add real steps one at a time — checkout → setup runtime → install → test → deploy. Breaking it down incrementally makes each piece easier to debug.
