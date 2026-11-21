# GitHub Actions Shared Library Architecture & Jenkins Mapping

Enterprise pattern documentation for reusable CI/CD workflows and shared utility actions in GitHub Actions across the `automationinferno` organization.

---

## 1. Jenkins Shared Libraries vs. GitHub Actions Shared Library

In Jenkins, a Shared Library is typically a Git repository containing Groovy scripts organized into `vars/` (global steps and pipeline definitions) and `src/` (object-oriented helper classes).

In GitHub Actions, there is no single monolithic "Groovy library" runtime. Instead, the architecture is split cleanly into two native layers:

| Jenkins Shared Library Construct | GitHub Actions Equivalent | Scope & Purpose |
| :--- | :--- | :--- |
| **Custom Pipeline Step (`vars/standardBuild.groovy`)** | **Composite Actions (`action.yml`)** | **Step-level reusable utilities**. Bundles runtime setup, caching, lockfile validation, Slack notifications, or step summaries. Can be invoked anywhere in any job. |
| **Pipeline Template (`vars/mavenPipeline.groovy`)** | **Reusable Workflows (`workflow_call`)** | **Job/Pipeline-level orchestration**. Standardizes full execution matrices, security posture (`permissions: {}`), timeout ceilings, and runner pools across an entire organization. |
| **Object Helper (`src/org/utils/Helper.groovy`)** | **Custom JavaScript Action (`@actions/core`) or CLI Script** | **Logic helpers**. Complex GitHub API queries, secret processing, or artifact checksum calculations. |
| **Import Directive (`@Library('my-lib@2.1') _`)** | **`uses: org/repo/...@v1` or `@<sha>`** | **Explicit version binding** per workflow or action reference. |

---

## 2. Organization Architecture Pattern

```text
automationinferno/gha-shared-library/
├── .github/
│   └── workflows/
│       ├── reusable-node-ci.yml          <-- Reusable Workflow (Node.js lint, test, build)
│       ├── reusable-docker-build-push.yml <-- Reusable Workflow (Buildx image build/push)
│       ├── reusable-helm-gitops-bump.yml  <-- Reusable Workflow (Helm values.yaml GitOps bump)
│       ├── reusable-release-drafter.yml   <-- Reusable Workflow (Release Drafter)
│       └── reusable-release-publish.yml   <-- Reusable Workflow (Release Publish)
├── actions/
│   ├── setup-node-env/                   <-- Composite Action (Custom Step: Node + cache + clean install)
│   │   └── action.yml
│   └── build-summary/                    <-- Composite Action (Custom Step: Writes $GITHUB_STEP_SUMMARY)
│       └── action.yml
└── README.md
```

### Layer 1: Reusable Workflows (`workflow_call`)
- **`.github/workflows/reusable-node-ci.yml`**: Standard Node.js CI (`lint` -> `test` -> `build`) with `permissions: {}` at root, `contents: read` on the job, and `timeout-minutes: 15`.
- **`.github/workflows/reusable-docker-build-push.yml`**: Builds and optionally pushes container images with Docker Buildx, optional QEMU cross-platform emulation, GitHub Actions Buildx cache (`type=gha`), and `digest` output (`contents: read` on caller job; `REGISTRY_USERNAME`/`REGISTRY_PASSWORD` secrets when `push: true`).
- **`.github/workflows/reusable-helm-gitops-bump.yml`**: Checks out a target GitOps repository using `GITOPS_TOKEN`, updates a validated dotted YAML key path (default `.image.tag`) in a Helm `values.yaml` via `yq`, and commits/pushes if changed (`contents: read` on caller job).
- **`.github/workflows/reusable-release-drafter.yml`**: Drafts release notes using the organization default config in `automationinferno/.github` (`contents: write`, `pull-requests: read` on caller job).
- **`.github/workflows/reusable-release-publish.yml`**: Orchestrates release publication hooks on published GitHub Releases (`contents: read`, `packages: write` on caller job).

### Layer 2: Composite Actions (The "vars/ Custom Steps")
- **`actions/setup-node-env`**: Standardizes `actions/setup-node`, npm caching keys, and lockfile execution (`npm ci`).
- **`actions/build-summary`**: Standardizes step summary markdown tables displayed natively in GitHub run logs.

---

## 3. Real-World Consumer Example

In consumer repository [`automationinferno/poc-node-app`](https://github.com/automationinferno/poc-node-app):

```yaml
name: Continuous Integration

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions: {}

jobs:
  call-node-ci:
    name: Run Node CI Pipeline
    permissions:
      contents: read
    uses: automationinferno/gha-shared-library/.github/workflows/reusable-node-ci.yml@v1
    with:
      node-version-file: '.nvmrc'
      run-lint: true
      run-test: true
      run-build: true
```

---

## 4. Key Lessons Learned & Guardrails

1. **Caller Job Permissions**:
   - When a caller workflow sets root `permissions: {}`, any calling job delegating to a reusable workflow **must** explicitly grant the necessary caller permissions (e.g. `permissions: contents: read`), otherwise GitHub Actions reports an immediate `startup_failure` before queuing runners.
2. **Action SHA Pinning**:
   - Pinned SHAs must reference exact commit SHAs (or valid tag references) in the target action repository.
3. **Repository Visibility**:
   - In GitHub Free / Team organizations, repositories hosting shared actions/workflows must be **public** to be referenced by other repositories without custom Personal Access Tokens or GitHub Apps. In GitHub Enterprise, **Internal** visibility is used to restrict sharing to the enterprise.
4. **Versioning Contracts**:
   - Use moving major tags like `@v1` for automatic non-breaking updates across consumer repos.
   - For high-security or regulated environments, pin consumer repos to immutable full commit SHAs.
