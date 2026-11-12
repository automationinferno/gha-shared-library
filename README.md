# GitHub Actions Shared Library (`gha-shared-library`)

Enterprise-style shared library repository for GitHub Actions across the `automationinferno` organization.

Analogous to a **Jenkins Shared Library**, this repository provides:
1. **Pipeline Templates (Reusable Workflows)**: Orchestrate standardized multi-step jobs with security baselines (`permissions: {}`, timeout controls, concurrency management).
2. **Custom Steps / Utils (Composite Actions)**: Granular, single-purpose utilities you can embed directly inside jobs across any repository.

---

## 1. Reusable Workflows

### `reusable-node-ci.yml`
Standard CI pipeline template for building, testing, and linting Node.js applications.

```yaml
jobs:
  ci:
    uses: automationinferno/gha-shared-library/.github/workflows/reusable-node-ci.yml@v1
    with:
      node-version: '20'
      run-lint: true
      run-test: true
      run-build: true
```

#### Inputs
| Input | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `node-version` | string | `'20'` | Node.js version |
| `node-version-file` | string | `''` | Path to `.nvmrc` or `.node-version` |
| `run-lint` | boolean | `true` | Executes `npm run lint` |
| `run-test` | boolean | `true` | Executes `npm test` |
| `run-build` | boolean | `true` | Executes `npm run build` |
| `working-directory` | string | `'.'` | Subdirectory path |

---

## 2. Shared Utilities (Composite Actions)

### `actions/setup-node-env`
Configures Node runtime, automatic npm dependency caching, and clean `npm ci` execution.

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: automationinferno/gha-shared-library/actions/setup-node-env@v1
    with:
      node-version: '20'
```

### `actions/build-summary`
Standardized Markdown step summary writer for GitHub Actions run logs.

```yaml
steps:
  - uses: automationinferno/gha-shared-library/actions/build-summary@v1
    with:
      title: 'Release Summary'
      test-result: 'success'
```

---

## 3. Versioning Strategy

- **`@v1`**: Major version pointer (recommended for consumers wanting seamless non-breaking updates).
- **`@v1.0.0`**: Immutable semantic release tag (for strictly controlled environments).
- **`@<commit-sha>`**: Cryptographic immutable pinning for zero-trust compliance.
