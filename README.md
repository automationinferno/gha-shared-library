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
│       └── reusable-node-ci.yml      <-- Reusable Workflow (Pipeline Template)
├── actions/
│   ├── setup-node-env/               <-- Composite Action (Custom Step: Node + cache + clean install)
│   │   └── action.yml
│   └── build-summary/                <-- Composite Action (Custom Step: Writes $GITHUB_STEP_SUMMARY)
│       └── action.yml
└── README.md
```

### Layer 1: Reusable Workflow (`workflow_call`)
Defined in `.github/workflows/reusable-node-ci.yml`:
- Acts as the central pipeline contract.
- Declares input parameters (`node-version`, `run-lint`, `run-test`, `run-build`).
- Enforces baseline security: `permissions: {}` at root, explicit scoped permissions per job, and execution `timeout-minutes`.
- Composes the step-level utilities from `actions/`.

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
