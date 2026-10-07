# CI/CD

Every chapter so far has added something to Beacon's pipeline: tests (Book I), PR checks
(Book II), contract checks (Book VII), image builds and scans (Book VIII), migrations (Book
IV). This chapter assembles them into a complete **continuous integration and continuous
delivery** pipeline: every change is built, tested, packaged once, and promoted through
environments to production automatically, safely and visibly.

CI/CD is where engineering practices turn into delivery speed. Teams that deploy small
changes many times a day have fewer, smaller incidents than teams that deploy big batches
monthly, not despite deploying often, but because of it.

---

## 1. The problem: the path from commit to production

Without automation, releasing software is a manual, error-prone ritual: someone builds on
their machine, runs some tests, copies files, runs migrations by hand, updates configuration,
and hopes. Releases become rare and large, so each one is risky, so they become rarer still.

CI/CD replaces the ritual with a pipeline: a repeatable, automated, observable path from a
commit to running in production.

---

## 2. The mental model

### CI, continuous delivery, continuous deployment

| Practice | Meaning |
|---|---|
| **Continuous Integration (CI)** | Every change is merged to the main branch frequently and verified automatically (build, tests, analysis) |
| **Continuous Delivery** | Every change that passes CI is **deployable** to production at the push of a button |
| **Continuous Deployment** | Every change that passes the pipeline **is deployed** to production automatically |

### The pipeline as a series of gates

```text
 commit/PR ─► CI: build · unit tests · integration tests · lint · contract checks · security scans
             │
             ▼ merge to main
           Build once: versioned artifacts (container images, migration bundle, IaC templates)
             │
             ▼
           Deploy to staging ─► migrations ─► smoke tests ─► E2E tests
             │
             ▼  (approval gate, or automatic)
           Deploy to production ─► migrations ─► canary/rolling ─► smoke tests ─► monitor SLOs
             │                                                          │ problems?
             └──────────────────────────────────────────────────────────┴► automatic rollback
```

Each gate catches a class of problems as early (and cheaply) as possible.

> **🧱 Durable:** **Build once, deploy many.** The exact artifact (image digest) that passed
> tests in staging is the one promoted to production. Rebuilding for each environment means
> production runs something that was never tested.

### DORA metrics

The *Accelerate* research (DORA) identified four metrics that distinguish high-performing
teams:

| Metric | Elite performance (indicative) |
|---|---|
| **Deployment frequency** | On demand, multiple times per day |
| **Lead time for changes** (commit → production) | Less than a day |
| **Change failure rate** | Low (roughly 0–15%) |
| **Time to restore service** | Less than an hour |

Speed and stability correlate positively: the practices that enable frequent deployment
(small changes, automation, fast feedback) also reduce failures and recovery time.

---

## 3. CI: fast, trustworthy feedback

A CI run should be **fast** (developers wait for it: aim for under ~10 minutes) and
**trustworthy** (green means good; red means a real problem).

Techniques:

- **Run stages in parallel**: backend build and tests, frontend lint/test/build, container
  builds, security scans.
- **Cache dependencies**: NuGet, pnpm store, Docker layers.
- **Test pyramid**: most tests fast; integration tests with Testcontainers; E2E tests in later
  stages or nightly (Book VII, Chapter 3).
- **Fail fast**: cheap checks (format, lint, compile) first.
- **No flaky tests** (Book VII, Chapter 3): quarantine and fix immediately.
- **Required status checks** on the main branch (Book II, Chapter 3).

### Security in the pipeline ("shift left")

- **Dependency scanning**: NuGet and npm vulnerability audits; Dependabot/Renovate updates.
- **Static analysis (SAST)**: CodeQL or equivalent.
- **Secret scanning** with push protection.
- **Container image scanning** (Book VIII, Chapter 6).
- **IaC scanning**: misconfigurations in Bicep/Terraform (Chapter 9).
- **SBOM and provenance** attached to images.

---

## 4. Pipeline security: the pipeline is production

A CI/CD system can deploy to production, so it's one of the most valuable targets for an
attacker. Protect it like production:

- **No long-lived cloud credentials** in CI: use OIDC workload identity federation (Chapter 2).
- **Least privilege per stage**: the PR build has no deploy permissions; only the deploy job
  for `environment: production` can obtain the production identity.
- **Protected environments**: required reviewers, branch restrictions, wait timers.
- **Pin third-party actions** to full commit SHAs (tags can be moved to malicious code), and
  limit which actions are allowed.
- **Least-privilege `GITHUB_TOKEN`** permissions per workflow.
- **Never run untrusted code with secrets**: workflows triggered by pull requests from forks
  must not have access to secrets (be careful with `pull_request_target`).
- **Audit** who changed pipeline definitions; require review for `.github/workflows/` via
  CODEOWNERS.

---

## 5. GitHub Actions and Azure DevOps

Both are capable CI/CD platforms; the concepts map directly:

| Concept | GitHub Actions | Azure Pipelines (Azure DevOps) |
|---|---|---|
| Definition | `.github/workflows/*.yml` | `azure-pipelines.yml` |
| Unit of work | Workflow → jobs → steps | Pipeline → stages → jobs → steps |
| Reusable logic | Actions, reusable workflows, composite actions | Tasks, templates |
| Runners | GitHub-hosted or self-hosted runners | Microsoft-hosted or self-hosted agents |
| Environments and approvals | Environments with protection rules | Environments with approvals and checks |
| Secrets | Repository/environment secrets, OIDC | Variable groups, Key Vault links, service connections (with workload identity) |
| Artifacts | Actions artifacts, GitHub Packages/GHCR | Pipeline artifacts, Azure Artifacts |

Azure DevOps also includes Boards (work tracking), Repos and Test Plans, and is common in
enterprises already invested in Microsoft tooling. GitHub is the default for code hosted on
GitHub. Beacon uses **GitHub Actions**; the same pipeline translates to Azure Pipelines
stage for stage.

---

## 6. Deploying databases and applications together

The order of operations matters (Book IV, Chapter 7; Book VII, Chapter 1):

1. **Migrations first**, and they must be **backward compatible** with the version currently
   running (expand step). The old app keeps working against the new schema.
2. **Deploy the new application version** (rolling or canary; Chapter 9).
3. **Contract step** (remove old columns) in a **later** release, once no running version
   uses them.

Migrations run as a separate job with the migrator identity, using an EF Core migration
bundle built in CI (an artifact, like the images):

```bash
dotnet ef migrations bundle -p src/Beacon.Infrastructure -s src/Beacon.Api \
  --self-contained -r linux-x64 -o artifacts/efbundle
# in the deploy job, as a Container Apps job or step with network access to the database:
./efbundle --connection "$MIGRATOR_CONNECTION_STRING"
```

---

## 7. In practice: Beacon's pipeline

### Workflow structure

```text
.github/workflows/
  ci.yml          on: pull_request, push to main    build, test, analyze (Books I–VII)
  release.yml     on: push to main                   build images + bundle once → deploy staging → E2E → deploy prod
  nightly.yml     on: schedule                       full E2E across browsers, image rescans, dependency audit
```

### `release.yml` (condensed)

```yaml
name: Release
on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write          # OIDC to Azure
  packages: write          # push images to GHCR

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      api-digest: ${{ steps.api.outputs.digest }}
      bff-digest: ${{ steps.bff.outputs.digest }}
    steps:
      - uses: actions/checkout@<pinned-sha>
      - uses: docker/setup-buildx-action@<pinned-sha>
      - uses: docker/login-action@<pinned-sha>
        with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }

      - id: api
        uses: docker/build-push-action@<pinned-sha>
        with:
          file: Dockerfile.api
          push: true
          tags: ghcr.io/awccon/beacon-api:${{ github.sha }}
          build-args: GIT_SHA=${{ github.sha }}
          sbom: true
          provenance: mode=max
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - id: bff
        uses: docker/build-push-action@<pinned-sha>
        with:
          file: Dockerfile.bff
          push: true
          tags: ghcr.io/awccon/beacon-bff:${{ github.sha }}
          build-args: GIT_SHA=${{ github.sha }}

      - name: Scan images
        run: |
          trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed ghcr.io/awccon/beacon-api@${{ steps.api.outputs.digest }}
          trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed ghcr.io/awccon/beacon-bff@${{ steps.bff.outputs.digest }}

      - uses: actions/setup-dotnet@<pinned-sha>
        with: { dotnet-version: '10.0.x' }
      - run: dotnet tool restore && dotnet ef migrations bundle -p src/Beacon.Infrastructure -s src/Beacon.Api --self-contained -r linux-x64 -o artifacts/efbundle
      - uses: actions/upload-artifact@<pinned-sha>
        with: { name: deploy, path: [artifacts/efbundle, infra/] }

  deploy-staging:
    needs: build
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
      api-image: ghcr.io/awccon/beacon-api@${{ needs.build.outputs.api-digest }}
      bff-image: ghcr.io/awccon/beacon-bff@${{ needs.build.outputs.bff-digest }}
    secrets: inherit

  e2e-staging:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@<pinned-sha>
      - run: pnpm --dir e2e install --frozen-lockfile && pnpm --dir e2e exec playwright install --with-deps chromium
      - run: pnpm --dir e2e exec playwright test --project=chromium
        env: { BASE_URL: https://staging.beacon.example.com }

  deploy-production:
    needs: [build, e2e-staging]
    uses: ./.github/workflows/deploy.yml
    with:
      environment: production          # protected: required reviewer, main branch only
      api-image: ghcr.io/awccon/beacon-api@${{ needs.build.outputs.api-digest }}
      bff-image: ghcr.io/awccon/beacon-bff@${{ needs.build.outputs.bff-digest }}
    secrets: inherit
```

### The reusable deploy workflow

```yaml
# .github/workflows/deploy.yml (condensed)
on:
  workflow_call:
    inputs: { environment: { type: string, required: true }, api-image: { type: string }, bff-image: { type: string } }

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/download-artifact@<pinned-sha>
        with: { name: deploy }
      - uses: azure/login@<pinned-sha>
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Infrastructure (Bicep, Chapter 9)
        run: az deployment group create -g ${{ vars.RESOURCE_GROUP }} -f infra/main.bicep -p infra/${{ inputs.environment }}.bicepparam

      - name: Database migrations (backward compatible)
        run: az containerapp job start -g ${{ vars.RESOURCE_GROUP }} -n job-beacon-migrate --image ${{ inputs.api-image }} --wait

      - name: Deploy API and BFF (new revisions, canary in production: Chapter 9)
        run: |
          az containerapp update -g ${{ vars.RESOURCE_GROUP }} -n ca-beacon-api --image ${{ inputs.api-image }}
          az containerapp update -g ${{ vars.RESOURCE_GROUP }} -n ca-beacon-bff --image ${{ inputs.bff-image }}

      - name: Smoke tests
        run: |
          curl --fail --retry 10 --retry-delay 6 --retry-all-errors https://${{ vars.HOSTNAME }}/health/ready
          curl --fail https://${{ vars.HOSTNAME }}/version | grep "${{ github.sha }}"

      - name: Annotate deployment in Application Insights
        run: az monitor app-insights events show ... # or post an annotation via REST: Chapter 6's dashboards
```

### What this pipeline guarantees

- **One build** produces immutable, scanned, signed-provenance images, deployed **by digest**
  to every environment.
- **No secrets** in GitHub for Azure: OIDC federation with environment-scoped identities.
- **Production requires** passing CI, staging deployment, staging E2E, and a reviewer's
  approval (removable later for full continuous deployment).
- **Migrations** run before the new code, as a job, with the migrator identity.
- **Smoke tests** verify the right version is live; failures stop the workflow (and, with
  Chapter 9's canary, trigger rollback).
- **Visibility**: every deployment is a workflow run linked to a commit, a PR and its review.

---

## 8. What can go wrong

- **Slow, flaky CI** that developers learn to ignore or bypass.
- **Rebuilding per environment**, deploying untested artifacts.
- **Long-lived cloud credentials** stored as CI secrets.
- **Unpinned third-party actions** and over-permissive tokens.
- **Migrations coupled to app startup**, or not backward compatible, causing outages during
  rollout.
- **Manual steps** "just this once" that drift from the pipeline.
- **No post-deployment verification**: green pipeline, broken production.
- **Big-batch releases** that make every deployment risky.

---

## 9. How an experienced engineer thinks about this

- **Small batches, deployed often**, are safer than big releases.
- **Build once, promote the same artifact.**
- **The pipeline is production infrastructure**: least privilege, review, pinned
  dependencies, no stored secrets.
- **Every manual step is a future incident**; automate it or document why not.
- **Measure delivery** with DORA metrics, and improve the slowest stage.

---

## 10. Check yourself

**Questions**

1. Distinguish continuous integration, continuous delivery and continuous deployment.
2. What does "build once, deploy many" mean, and why does it matter?
3. Name the four DORA metrics. Why do speed and stability correlate?
4. What makes CI fast and trustworthy?
5. Why is the CI/CD system a high-value target, and how do you protect it?
6. In what order should migrations and application deployments happen, and why?
7. What should post-deployment smoke tests check?

**Exercises**

1. Build Beacon's `release.yml` with image build, scan and staging deployment.
2. Configure GitHub environments `staging` and `production` with protection rules and
   environment-scoped OIDC identities.
3. Pin every third-party action in Beacon's workflows to a commit SHA and enable Dependabot
   for GitHub Actions.
4. Measure your team's (or Beacon's) four DORA metrics for a month.

**Interview-style questions**

- "Describe a CI/CD pipeline you've built or would build."
- "How do you deploy database changes safely?"
- "How do you secure a CI/CD pipeline?"

---

## 11. Going deeper

- Nicole Forsgren, Jez Humble, Gene Kim, *Accelerate*.
- Jez Humble, David Farley, *Continuous Delivery*.
- [GitHub Actions: Security hardening](https://docs.github.com/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [Azure Pipelines documentation](https://learn.microsoft.com/azure/devops/pipelines/)

**Next:** [Chapter 9 — Infrastructure as Code and Deployment Strategies](09-infrastructure-as-code-and-deployment-strategies.md)
defines Beacon's infrastructure in code and deploys it without downtime.
