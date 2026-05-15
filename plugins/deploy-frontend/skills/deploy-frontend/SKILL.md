---
name: deploy-frontend
description: >
  Deploy the RWA Frontend and/or Chatbot API to Azure Container Apps.
  Primary method is GitHub CI/CD: push to `develop` deploys staging, merge
  `develop` → `main` deploys production. Manual Docker build is the emergency
  fallback only.

  Trigger this skill when the user says "deploy", "ship it", "push to
  staging", "push to prod", "release this", "promote develop to main", or
  asks to verify a deployment / check staging.
---

# Deploy to Azure

## Branch model — quick reference

| Branch | Environment | Container Apps |
|--------|-------------|----------------|
| `develop` | **Staging** | `rwa-frontend-staging`, `rwa-chatbot-api-staging` |
| `main` | **Production** | `rwa-frontend`, `rwa-chatbot-api` |

Pushing to either branch fires `.github/workflows/deploy.yml`, which builds Docker images on GitHub runners, pushes to ACR (`rwatableauportalacr`), and updates the matching Container Apps.

---

## Standard workflow (CI/CD — always prefer this)

1. **Branch from `develop`** and make changes.
2. **Push and open a PR into `develop`**. The PR Quality Gate workflow runs (lint, type-check, Docker build verification).
3. **Merge the PR** — GitHub Actions builds, pushes to ACR, deploys to **staging**.
4. **Verify on staging:** `https://rwa-frontend-staging.happyforest-c0f9b16c.northeurope.azurecontainerapps.io`.
5. **Open a PR `develop` → `main`** and merge — deploys to **production**.

### Direct push to develop (when bypassing PR is appropriate)

```powershell
git push origin develop
```

Use this for: hotfixes, audit-loop hardening commits already reviewed, follow-up tweaks to a just-merged PR. Otherwise prefer PRs so the Quality Gate runs.

### Force-deploy everything (manual workflow trigger)

```powershell
gh workflow run deploy.yml --repo Real-World-Analytics/RWA-Frontend --field force_deploy=true
```

---

## What this means for the agent

- After making code changes, **commit and push to `develop`** (or the current feature branch + PR). **Do not** run a local Docker build.
- When the user says "deploy", "ship it", or "push to staging", push the changes to `develop`. For "push to prod" or "promote", open or merge a PR from `develop` to `main`.
- If CI fails, inspect `.github/workflows/deploy.yml` and GitHub secrets first. Don't fall back to local Docker until that's exhausted.

### When to skip deployment entirely

- Markdown-only edits, `.cursor/`/`.claude/` rules, comments-only changes, or anything with no runtime effect.
- Failing tests in the worktree.
- Local untracked files that haven't been reviewed.

---

## Verify deployment

```powershell
# Most recent workflow runs
gh run list --repo Real-World-Analytics/RWA-Frontend --workflow deploy.yml --limit 3

# Live revision name (staging)
az containerapp show --name rwa-frontend-staging `
  --resource-group rg_rwatableauportal-prod `
  --query "properties.latestRevisionName" -o tsv

# Live revision name (production)
az containerapp show --name rwa-frontend `
  --resource-group rg_rwatableauportal-prod `
  --query "properties.latestRevisionName" -o tsv
```

## Environment URLs

**Staging**
- Frontend: `https://rwa-frontend-staging.happyforest-c0f9b16c.northeurope.azurecontainerapps.io`
- Chatbot:  `https://rwa-chatbot-api-staging.happyforest-c0f9b16c.northeurope.azurecontainerapps.io`

**Production**
- Frontend: `https://rwa-frontend.happyforest-c0f9b16c.northeurope.azurecontainerapps.io`
- Chatbot:  `https://rwa-chatbot-api.happyforest-c0f9b16c.northeurope.azurecontainerapps.io`

---

## Emergency fallback — manual Docker build

**Only use this if GitHub Actions is broken** (expired credentials, runner outage, ACR auth failure, etc.). Normal workflow is CI/CD — do not deploy manually for convenience.

### Prerequisites

- Docker Desktop running locally (BuildKit enabled — default since Docker 23)
- Azure CLI (`az`) logged in to the subscription containing `rg_rwatableauportal-prod`
- Environment variables `DATABASE_URL` and `NEXTAUTH_SECRET` exported in your shell (read from `.env` or Key Vault — never paste them into the command line, they end up in shell history)

### Steps

```powershell
az acr login --name rwatableauportalacr
```

### Frontend (matches the CI's BuildKit-secret pattern)

```powershell
$env:DOCKER_BUILDKIT = "1"
# DATABASE_URL and NEXTAUTH_SECRET must already be in the shell env.
docker build -f Dockerfile.frontend `
  --secret id=DATABASE_URL,env=DATABASE_URL `
  --secret id=NEXTAUTH_SECRET,env=NEXTAUTH_SECRET `
  -t rwatableauportalacr.azurecr.io/rwa-frontend:latest .

docker push rwatableauportalacr.azurecr.io/rwa-frontend:latest

# Pin the digest from the push output, then update the Container App
az containerapp update --name rwa-frontend `
  --resource-group rg_rwatableauportal-prod `
  --image "rwatableauportalacr.azurecr.io/rwa-frontend@sha256:<digest>" `
  --set-env-vars "RESTART_TRIGGER=$(Get-Date -UFormat %s)"
```

For **staging**, change the target Container App to `rwa-frontend-staging`.

### Chatbot API

```powershell
docker build -t rwatableauportalacr.azurecr.io/rwa-chatbot-api:latest ./backend-services/chatbot
docker push rwatableauportalacr.azurecr.io/rwa-chatbot-api:latest

az containerapp update --name rwa-chatbot-api `
  --resource-group rg_rwatableauportal-prod `
  --image "rwatableauportalacr.azurecr.io/rwa-chatbot-api@sha256:<digest>" `
  --set-env-vars "RESTART_TRIGGER=$(Get-Date -UFormat %s)"
```

**Critical:** never use `--build-arg DATABASE_URL=...` or `--build-arg NEXTAUTH_SECRET=...`. Those values get baked into image layers and are recoverable by anyone with the image. BuildKit `--secret` mounts make them available during `next build` without writing them to disk.

---

## Azure resource reference

| Resource | Production | Staging |
|----------|-----------|---------|
| Frontend App | `rwa-frontend` | `rwa-frontend-staging` |
| Chatbot App | `rwa-chatbot-api` | `rwa-chatbot-api-staging` |
| ACR | `rwatableauportalacr` | same |
| Resource Group | `rg_rwatableauportal-prod` | same |
| ACA Environment | `cae-rwa-portal-vnet` | same |
| Dockerfile (frontend) | `Dockerfile.frontend` (repo root) | same |
| Dockerfile (chatbot) | `backend-services/chatbot/Dockerfile` | same |

## GitHub secrets required by CI

`AZURE_CREDENTIALS`, `ACR_USERNAME`, `ACR_PASSWORD`, `DATABASE_URL`, `NEXTAUTH_SECRET`, `ANTHROPIC_API_KEY` (chatbot verifier), `CHATBOT_VERIFY_ADMIN_KEY` (chatbot post-deploy verification).

## CI/CD service credentials

- Service principal: **RWA Tableau Deployment App** (`appId: eec4ae35-734e-4c4b-b211-dcb2052d73cf`)
- Role: Contributor on `rg_rwatableauportal-prod`
- Secret expires: April 2028

---

## Post-deploy verification (chatbot only — runs automatically)

The deploy workflow includes a `verify-chatbot` job that polls `/health`, runs canary queries against the chatbot, and:
- On `develop` → soft-warn (logs failures, never blocks)
- On `main` → hard-fail with auto-revert to the previous image

See `backend-services/chatbot/scripts/ci/verify_deployed_chatbot.py` for the canary set.
