# Harness GitHub Actions

Official composite actions for **Harness Software Supply Chain Assurance (SSCA)** in GitHub workflows: SBOM orchestration, SLSA provenance, policy enforcement, and artifact signing—all backed by the `harness/ssca-plugin` container.

[![Documentation](https://img.shields.io/badge/Docs-Harness%20SSCA-blue?logo=read-the-docs)](https://developer.harness.io/docs/software-supply-chain-assurance)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-green.svg)](https://github.com/harness/github-actions/blob/main/LICENSE.md)

---

## Contents

- [Available actions](#available-actions)
- [Quick start](#quick-start)
- [Artifact source types](#artifact-source-types)
- [Signing methods](#signing-methods)
- [Inputs: action `with:` vs job `env`](#inputs-action-with-vs-job-env)
- [Shared Harness inputs](#shared-harness-inputs)
- [Action reference](#action-reference)
- [Sample workflows](#sample-workflows)

---

## Available actions

Each action is a **root-level composite** in this repository (for example `sbom-generation/action.yml`). Reference them from your workflow as `harness/github-actions/<name>@<ref>`.

| Capability | Marketplace path | Plugin command |
|------------|------------------|----------------|
| SBOM generation (orchestrate) | [`sbom-generation`](sbom-generation/action.yml) | `github-orchestrate` |
| SBOM ingestion | [`sbom-ingestion`](sbom-ingestion/action.yml) | `github-orchestrate` |
| SBOM verify + policy enforce | [`sbom-policy-enforcement`](sbom-policy-enforcement/action.yml) | `github-enforce` |
| SLSA provenance (attest) | [`slsa-generation`](slsa-generation/action.yml) | `github-attest` |
| SLSA verification | [`slsa-verification`](slsa-verification/action.yml) | `github-verify` |
| Artifact signing | [`artifact-signing`](artifact-signing/action.yml) | `github-sign` |
| Artifact verification | [`artifact-verification`](artifact-verification/action.yml) | `github-verify` |

Internal helper (not intended for direct use in customer workflows): [`run-plugin-container`](run-plugin-container/action.yml) — OIDC token fetch, env filtering, and `docker run` for the pinned plugin image.

**Plugin image:** `harness/ssca-plugin:0.69.0` (fixed inside each action).

**Version pin:** use a release tag (for example `@1.2.0`) or `@main` while validating; avoid floating refs in production.

---

## Quick start

Minimal **container image** flow with Vault signing:

```yaml
permissions:
  contents: read

jobs:
  ssca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: SBOM generation
        uses: harness/github-actions/sbom-generation@main
        with:
          HARNESS_ACCOUNT_URL: ${{ vars.HARNESS_ACCOUNT_URL }}
          HARNESS_API_KEY: ${{ secrets.HARNESS_API_KEY }}
          HARNESS_ACCOUNT_ID: ${{ vars.HARNESS_ACCOUNT_ID }}
          HARNESS_ORG_ID: ${{ vars.HARNESS_ORG_ID }}
          HARNESS_PROJECT_ID: ${{ vars.HARNESS_PROJECT_ID }}
          SBOM_TOOL: Syft
          SBOM_FORMAT: cyclonedx-json
          SOURCE_TYPE: docker
          SBOM_SOURCE: myregistry.io/myapp:1.0.0
          ATTEST_SBOM: true
          VAULT_ADDR: ${{ secrets.VAULT_ADDR }}
          VAULT_TOKEN: ${{ secrets.VAULT_TOKEN }}
          KMS_KEY: my_cosign_transit_key
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

---

## Artifact source types

Set **`SOURCE_TYPE`** on every action that targets an artifact. Use the matching **source input** (`SBOM_SOURCE`, `SLSA_SOURCE`, or `SIGNING_SOURCE`).

| `SOURCE_TYPE` | Typical use | `SBOM_SOURCE` / `SLSA_SOURCE` / `SIGNING_SOURCE` | Extra inputs |
|---------------|-------------|---------------------------------------------------|--------------|
| **`docker`** (default) | Container image in a registry | Image reference, e.g. `docker.io/library/nginx:1.25` | `DOCKER_USERNAME`, `DOCKER_PASSWORD` when the registry is private |
| **`acr`**, **`gcr`**, **`ecr`**, **`gar`**, **`har`** | Cloud / Harbor registries | Same as docker — full image URI | Registry auth via `DOCKER_*` or your registry login step before the action |
| **`local`** | Non-container artifact (zip, tarball, binary) on the runner | Path to the file, e.g. `${{ github.workspace }}/artifacts/app.zip` | **`ARTIFACT_WORKSPACE`**, **`ARTIFACT_NAME`**, **`ARTIFACT_VERSION`** |
| **`repository`** | SBOM / SLSA from source code | Git URL, e.g. `https://github.com/org/service.git` | Set **job `env`**: `REPO_URL`, `REPO_PATH`, `REPO_VARIANT`, `REPO_VARIANT_TYPE`, `REPO_CLONED_CODEBASE` (see below) |

### Container (`SOURCE_TYPE: docker`)

```yaml
- uses: harness/github-actions/sbom-generation@main
  with:
    SOURCE_TYPE: docker
    SBOM_SOURCE: ${{ vars.TARGET_IMAGE }}
    SBOM_TOOL: Syft
    SBOM_FORMAT: cyclonedx-json
    PLUGIN_SBOMDESTINATION: ${{ github.workspace }}/sbom-out
    # ... Harness + signing fields
```

The runner must reach the registry; log in with `docker/login-action` (or equivalent) before the step when credentials are required.

### Local / non-container (`SOURCE_TYPE: local`)

```yaml
- uses: harness/github-actions/sbom-generation@main
  with:
    SOURCE_TYPE: local
    SBOM_SOURCE: ${{ github.workspace }}/artifacts/scs-sample-app.zip
    ARTIFACT_WORKSPACE: ${{ github.workspace }}/artifacts/scs-sample-app.zip
    ARTIFACT_NAME: scs-sample-app.zip
    ARTIFACT_VERSION: v1
    SBOM_TOOL: Syft
    SBOM_FORMAT: cyclonedx-json
    PLUGIN_SBOMDESTINATION: ${{ github.workspace }}/sbom-out
    # ... Harness + signing fields
```

Use a path under **`GITHUB_WORKSPACE`** so the plugin container can read the file (the workspace is mounted into the container).

### Repository (`SOURCE_TYPE: repository`)

```yaml
jobs:
  ssca:
    runs-on: ubuntu-latest
    env:
      REPO_URL: https://github.com/myorg/my-service.git
      REPO_PATH: /
      REPO_VARIANT: main
      REPO_VARIANT_TYPE: branch
      REPO_CLONED_CODEBASE: ${{ github.workspace }}/checkout
    steps:
      - uses: actions/checkout@v4
        with:
          repository: myorg/my-service
          path: checkout

      - uses: harness/github-actions/sbom-generation@main
        with:
          SOURCE_TYPE: repository
          SBOM_SOURCE: https://github.com/myorg/my-service.git
          SBOM_TOOL: Syft
          SBOM_FORMAT: cyclonedx-json
          # ... Harness + signing fields
```

Repository metadata is passed via **job `env`**, not action inputs; `run-plugin-container` forwards those variables into the plugin.

---

## Signing methods

Attestation and signing steps support **HashiCorp Vault (Cosign KMS)** or **Harness Fulcio keyless** via GitHub OIDC.

| Method | When to use | Workflow / action setup |
|--------|-------------|-------------------------|
| **Vault KMS** | Production cosign keys in Vault Transit | `with:` → `KMS_KEY`, `VAULT_ADDR`, `VAULT_TOKEN` (often from secrets). Do **not** set `OIDC_PROVIDER`. |
| **Keyless (Fulcio)** | No long-lived signing key; GitHub-issued OIDC | Workflow `permissions: id-token: write`. Job `env: FULCIO_URL: <Harness Fulcio gateway>`. `with:` → `OIDC_PROVIDER: non-harness`. Set `ATTEST_SBOM` / `ATTEST_SLSA` as needed. |

### Vault KMS example

```yaml
- uses: harness/github-actions/slsa-generation@main
  with:
    SOURCE_TYPE: docker
    SLSA_SOURCE: myregistry.io/myapp:1.0.0
    ATTEST_SLSA: true
    VAULT_ADDR: ${{ secrets.VAULT_ADDR }}
    VAULT_TOKEN: ${{ secrets.VAULT_TOKEN }}
    KMS_KEY: SSCA_COSIGN_KEY
    # ... Harness account fields
```

### Keyless OIDC example

```yaml
permissions:
  contents: read
  id-token: write

env:
  FULCIO_URL: https://qa.harness.io/gateway/harness-fulcio/

jobs:
  ssca:
    runs-on: ubuntu-latest
    steps:
      - uses: harness/github-actions/sbom-generation@main
        with:
          SOURCE_TYPE: docker
          SBOM_SOURCE: myregistry.io/myapp:1.0.0
          ATTEST_SBOM: true
          OIDC_PROVIDER: non-harness
          SBOM_TOOL: Syft
          SBOM_FORMAT: cyclonedx-json
          # ... Harness account fields
```

Keyless signing **requires** `id-token: write` on the calling workflow; otherwise the action fails with a clear OIDC permissions error.

---

## Inputs: action `with:` vs job `env`

| Set on | Examples | Purpose |
|--------|----------|---------|
| Action **`with:`** | `HARNESS_*`, `SOURCE_TYPE`, `SBOM_SOURCE`, `SLSA_SOURCE`, `SIGNING_SOURCE`, `SBOM_TOOL`, `SBOM_FORMAT`, `PLUGIN_SBOMDESTINATION`, `ATTEST_SBOM`, `VERIFY_SBOM`, `ATTEST_SLSA`, `VERIFY_SLSA`, `POLICY_SET_REF`, `KMS_KEY`, `VAULT_*`, `OIDC_PROVIDER`, `ARTIFACT_*`, `DOCKER_*` | Public contract of each composite action |
| Job **`env:`** | `FULCIO_URL`, `REPO_URL`, `REPO_PATH`, `REPO_VARIANT`, `REPO_VARIANT_TYPE`, `REPO_CLONED_CODEBASE` | Forwarded into the plugin container; not declared as action inputs |

Empty optional values are stripped before the container starts so the plugin does not receive blank env vars.

---

## Shared Harness inputs

Required on **every** action:

| Input | Description |
|-------|-------------|
| `HARNESS_ACCOUNT_URL` | Harness account URL (e.g. `https://app.harness.io`) |
| `HARNESS_ACCOUNT_ID` | Account identifier |
| `HARNESS_ORG_ID` | Organization identifier |
| `HARNESS_PROJECT_ID` | Project identifier |
| `HARNESS_API_KEY` | API key (store in GitHub Secrets) |

---

## Action reference

### SBOM generation — `sbom-generation`

Generates an SBOM and uploads orchestration results to Harness. Optional SBOM attestation when `ATTEST_SBOM: true`.

| Input | Required | Notes |
|-------|----------|-------|
| `SBOM_TOOL` | Yes | e.g. `Syft`, `cdxgen` |
| `SBOM_FORMAT` | Yes | e.g. `cyclonedx-json`, `spdx-json` |
| `SOURCE_TYPE` | No | Default `docker`; see [artifact types](#artifact-source-types) |
| `SBOM_SOURCE` | No* | Image, file path, or repo URL |
| `PLUGIN_SBOMDESTINATION` | No | Output directory/path for generated SBOM files |
| `ATTEST_SBOM` | No | Default `false` |
| Signing | No | Vault or keyless — see [signing methods](#signing-methods) |

[Harness docs — Generate SBOM with GitHub Actions](https://developer.harness.io/docs/software-supply-chain-assurance/sbom/generate-sbom-with-github-actions)

---

### SBOM ingestion — `sbom-ingestion`

Ingests an existing SBOM file for an artifact. Sets `PLUGIN_MODE: ingestion`.

| Input | Required | Notes |
|-------|----------|-------|
| `PLUGIN_SBOMDESTINATION` | Yes | Path to the SBOM file to ingest |
| `SOURCE_TYPE`, `SBOM_SOURCE` | No | Target artifact identity |
| `ATTEST_SBOM` | No | Attest after ingest |

[Harness docs — Ingest SBOM](https://developer.harness.io/docs/software-supply-chain-assurance/sbom/ingest-sbom-with-github-actions)

---

### SBOM policy enforcement — `sbom-policy-enforcement`

Verifies SBOM attestation (optional) and enforces a Harness policy set.

| Input | Required | Notes |
|-------|----------|-------|
| `VERIFY_SBOM` | Yes | `true` to verify attestation before enforce |
| `POLICY_SET_REF` | Yes | Harness policy set reference |
| `SOURCE_TYPE`, `SBOM_SOURCE` | No | Artifact under test |

[Harness docs — Enforce SBOM policies](https://developer.harness.io/docs/software-supply-chain-assurance/sbom-policies/enforce-sbom-policies-with-gitHub-actions)

---

### SLSA generation — `slsa-generation`

Generates and optionally attests SLSA provenance (`github-attest`).

| Input | Required | Notes |
|-------|----------|-------|
| `SLSA_SOURCE` | No* | Image, file, or repo |
| `SOURCE_TYPE` | No | Default `docker` |
| `ATTEST_SLSA` | No | Default `false` |

[Harness docs — Generate SLSA](https://developer.harness.io/docs/software-supply-chain-assurance/slsa/generate-slsa-with-github-actions)

---

### SLSA verification — `slsa-verification`

Verifies SLSA provenance attestation for the target artifact.

| Input | Required | Notes |
|-------|----------|-------|
| `VERIFY_SLSA` | Yes | Enable verification |
| `SLSA_SOURCE`, `SOURCE_TYPE` | No | Artifact identity |

[Harness docs — Verify SLSA](https://developer.harness.io/docs/software-supply-chain-assurance/slsa/verify-slsa-with-github-actions)

---

### Artifact signing — `artifact-signing`

Signs the artifact with Cosign (`github-sign`).

| Input | Required | Notes |
|-------|----------|-------|
| `SIGNING_SOURCE` | No* | Image, file, or repo |
| `SOURCE_TYPE` | No | Default `docker` |
| Vault or keyless | No | Same as [signing methods](#signing-methods) |

---

### Artifact verification — `artifact-verification`

Verifies artifact signatures stored in the registry or associated with the artifact.

| Input | Required | Notes |
|-------|----------|-------|
| `SIGNING_SOURCE`, `SOURCE_TYPE` | No | Artifact identity |
| Vault or keyless | No | Verification key / OIDC context |

---

## Sample workflows

End-to-end examples (container, local, and repository × Vault vs keyless) live in the community sample repo:

**[KartikSaurya/ssca-github-actions](https://github.com/KartikSaurya/ssca-github-actions)** — workflows such as `container-keyless.yml`, `non-container-kms.yml`, and `repo-keyless.yml` call these marketplace actions directly.

---

## License

See [LICENSE.md](LICENSE.md). Usage of Harness services is subject to your Harness account terms.
