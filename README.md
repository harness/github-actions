# Harness GitHub Actions
GitHub Actions to integrate Harness capabilities with GitHub Workflows.

[![Documentation](https://img.shields.io/badge/Docs-View-blue?logo=read-the-docs)](https://developer.harness.io/)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-green.svg)](https://github.com/harness/github-actions/blob/main/LICENSE.md)


This GitHub Action provides a set of sub-actions for:

- [SBOM Generation and Attestation](#sbom-generation-and-attestation)
- [SBOM Ingestion and Attestation](#sbom-ingestion-and-attestation)
- [SBOM Verification and Policy Enforcement](#sbom-verification-and-policy-enforcement)
- [SLSA Provenance Generation and Attestation](#slsa-provenance-generation-and-attestation)
- [SLSA Provenance Verification](#slsa-provenance-verification)

All operations performed by these sub-actions save generated artifacts and details to Harness.

---

## Usage of the Actions

| **Features**                               | **Usage of Sub Actions**                                      |
|------------------------------------------|------------------------------------------------|
| [SBOM Generation and Attestation](#sbom-generation-and-attestation)          | `harness/github-actions/sbom-generation` |
| [SBOM Ingestion and Attestation](#sbom-ingestion-and-attestation)            | `harness/github-actions/sbom-ingestion`  |
| [SBOM Verification and Policy Enforcement](#sbom-verification-and-policy-enforcement)   | `harness/github-actions/sbom-policy-enforcement` |
| [SLSA Provenance Generation and Attestation](#slsa-provenance-generation-and-attestation)             | `harness/github-actions/slsa-generation` |
| [SLSA Provenance Verification](#slsa-provenance-verification)               | `harness/github-actions/slsa-verification` |

Each of these actions has its own configurations. Refer to the specific sections for detailed instructions and examples.

---

## Requirements

To use any of the sub-actions and perform operations successfully, you need the following:

1. **Harness Account**: Ensure you have a Harness account with the appropriate license enabled for the module to which the GitHub Action belongs.

2. **Harness Account Details**: Save the following Harness account details, which are required for all sub-actions. It is recommended to securely store these values using GitHub Secrets.

| **Key**                 | **Value Example**                           | **Description**                                                      | **Required** |
|--------------------------|---------------------------------------------|----------------------------------------------------------------------|-------------|
| `HARNESS_ACCOUNT_URL`   | `https://example.harness.io`               | The URL of your Harness account.                                     | Yes         |
| `HARNESS_ACCOUNT_ID`    | `ppdfedDDDL_dharzdPs_JtWT7g`               | The unique identifier for your Harness account.                      | Yes         |
| `HARNESS_ORG_ID`        | `SCS`                                       | The identifier for your Harness organization.                        | Yes         |
| `HARNESS_PROJECT_ID`    | `SCS_ORG`                                  | The identifier for your Harness project within the organization.     | Yes         |
| `HARNESS_API_KEY`       | `${{ secrets.SCS_API_KEY }}`                | The API key for authenticating with Harness. Create an API key using a Service Account (recommended) or a Personal Account , and then add the key to GitHub Actions Secrets with "HARNESS_API_KEY" as the key name. | Yes         |
| `VAULT_ADDR`   | `https://myvault.example.com`               | The URL of your Vault.	                                     | No         |

3. **Security Keys**: For attestation generation and verification, Key pair is required. The key should be generated using Cosign of type `ecdsa-P256`. Currently, HashiCorp Vault is supported for storing and retrieving the key. Additional Key Management Services (KMS) will be supported in the future.

---

## SBOM Generation and Attestation
> harness/github-actions/sbom-generation@1.1.0

The sub-action `harness/github-actions/sbom-generation` is responsible for generating the Software Bill of Materials (SBOM) and optionally attesting it. The generated SBOM is saved to Harness and can be found in the **Artifacts** section of the **SCS (Software Supply Chain Security)** module. If attestation is enabled, the SBOM attestation will be signed, and the `.att` attestation file will be pushed to the configured container registry. For more details, refer to [Harness documentation](https://developer.harness.io/docs/software-supply-chain-assurance/sbom/generate-sbom-with-github-actions).

**Note:** The signing key must be Cosign generated and stored in **HashiCorp Vault**. Currently, only HashiCorp Vault is supported, but more Key Management Systems (KMS) will be supported in the near future.

### Usage Example

```yaml
- name: SBOM Generation
  uses: harness/github-actions/sbom-generation@1.1.0
  with:
    HARNESS_ACCOUNT_URL: https://myaccount.harness.io
    HARNESS_ACCOUNT_ID: my_account_id_9YpRharzPs
    HARNESS_ORG_ID: my_org_id_default
    HARNESS_PROJECT_ID: example_project_id
    HARNESS_API_KEY: ${{ secrets.API_KEY_SAVED_AS_GH_SECRET }}
    VAULT_ADDR: ${{ secrets.VAULT_URL }}
    TARGET: example_image:image_tag
    TOOL: Syft
    FORMAT: spdx-json
    ATTEST: true
    KMS_KEY: path_to_my_key_in_vault
```

### Configuration

Make sure to include the required configurations from the [Requirements](#requirements) section in your workflow. Below are the specific configurations for the `harness/github-actions/sbom-generation` sub-action.

| **Key**         | **Value Example**       | **Description**                                            | **Required** |
|-----------------|-------------------------|------------------------------------------------------------|-------------|
| `TOOL`         | `Syft` or `cdxgen`        | The tool used to generate the SBOM.                        | Yes         |
| `FORMAT`       | `spdx-json` or `cyclonedx`| The format of the generated SBOM.                          | Yes         |
| `TARGET`       | `image_name:image_tag`  | The target artifact (Docker image) for SBOM generation.    | Yes         |
| `ATTEST`       | `true` or `false`         | Boolean flag to determine if attestation is required.      | No          |
| `KMS_KEY`      | `path/to/my/key`        | Path to the Private key used for signing the attestation.          | No          |

## SBOM Ingestion and Attestation
> harness/github-actions/sbom-ingestion@1.1.0

The sub-action `harness/github-actions/sbom-ingestion` is responsible for ingesting an existing SBOM and optionally attesting it. The SBOM will be saved to Harness. If attestation is enabled, the SBOM attestation will be signed, and the .att file will be pushed to the configured container registry.  For more details, refer to [Harness documentation](https://developer.harness.io/docs/software-supply-chain-assurance/sbom/ingest-sbom-with-github-actions).

**Note:** The signing key must be Cosign generated and stored stored in **HashiCorp Vault**. Currently, only HashiCorp Vault is supported, but more Key Management Systems (KMS) will be supported in the near future.

### Usage Example

```yaml
- name: SBOM Ingestion
  uses: harness/github-actions/sbom-ingestion@1.1.0
  with:
    HARNESS_ACCOUNT_URL: https://myaccount.harness.io
    HARNESS_ACCOUNT_ID: my_account_id_9YpRharzPs
    HARNESS_ORG_ID: my_org_id_default
    HARNESS_PROJECT_ID: example_project_id
    HARNESS_API_KEY: ${{ secrets.API_KEY_SAVED_AS_GH_SECRET }}
    VAULT_ADDR: ${{ secrets.VAULT_URL }}
    TARGET: <image>:<tag>
    SBOM_FILE_PATH: <path_to_sbom_file>
    ATTEST: true
    KMS_KEY: <path_to_your_key>
```

### Configuration

Make sure to include the required configurations from the [Requirements](#requirements) section in your workflow. Below are the specific configurations for the `harness/github-actions/sbom-ingestion` sub-action.

| **Key**           | **Value Example**         | **Description**                                               | **Required** |
|-------------------|---------------------------|---------------------------------------------------------------|-------------|
| `TARGET`          | `example_image:latest`    | The target artifact (Docker image) for SBOM ingestion.        | Yes         |
| `SBOM_FILE_PATH`  | `path/to/sbom.json`       | The file path to the SBOM that needs to be ingested.          | Yes         |
| `ATTEST`          | `true` or `false`           | Boolean flag to determine if attestation is required.         | No          |
| `KMS_KEY`         | `path/to/your/key`        | Path to the Private key used for signing the attestation.             | No          |


## SBOM Verification and Policy Enforcement
> harness/github-actions/sbom-policy-enforcement@1.1.0

The sub-action `harness/github-actions/sbom-policy-enforcement` verifies the SBOM attestation and enforces policies on the SBOM. The policies applied are **Harness SBOM Policies**. For more information on creating and managing SBOM policies, refer to the [Harness SBOM Policies Documentation](https://developer.harness.io/docs/software-supply-chain-assurance/sbom-policies/create-sbom-policies). For more details, refer to [Harness documentation](https://developer.harness.io/docs/software-supply-chain-assurance/sbom-policies/enforce-sbom-policies-with-gitHub-actions).

### Usage Example

```yaml
- name: SBOM Policy Enforcement
  uses: harness/github-actions/sbom-policy-enforcement@1.1.0
  with:
    HARNESS_ACCOUNT_URL: https://myaccount.harness.io
    HARNESS_ACCOUNT_ID: my_account_id_9YpRharzPs
    HARNESS_ORG_ID: my_org_id_default
    HARNESS_PROJECT_ID: example_project_id
    HARNESS_API_KEY: ${{ secrets.API_KEY_SAVED_AS_GH_SECRET }}
    VAULT_ADDR: ${{ secrets.VAULT_URL }}
    TARGET: example_image:latest
    VERIFY: true
    POLICY_SET_REF: github_opa_policy
    KMS_KEY: path/to/your/key
```

### Configuration

Make sure to include the required configurations from the [Requirements](#requirements) section in your workflow. Below are the specific configurations for the `harness/github-actions/sbom-policy-enforcement` sub-action.

| **Key**            | **Value Example**            | **Description**                                                                 | **Required** |
|--------------------|------------------------------|---------------------------------------------------------------------------------|-------------|
| `TARGET`           | `example_image:latest`       | The target artifact (Docker image) for SBOM verification and policy enforcement.| Yes         |
| `VERIFY`           | `true` or `false`              | Boolean flag to determine if attestation verification is required.              | Yes         |
| `POLICY_SET_REF`   | `github_opa_policy`          | The reference to the Harness SBOM policy set to be enforced.                    | Yes         |
| `KMS_KEY`          | `path/to/your/key`           | Path to the Public key used for verifying the attestation.                             | Yes          |

---

## SLSA Provenance Generation and Attestation
> harness/github-actions/slsa-generation@1.1.0

The sub-action `harness/github-actions/slsa-generation` is responsible for generating SLSA provenance and optionally attesting it. If attestation is enabled, the `.att` attestation file will be pushed to the configured container registry. The generated SLSA provenance can be found in the **Artifacts** section of the Harness **SCS (Software Supply Chain Security)** module. For more details, refer to [Harness documentation](https://developer.harness.io/docs/software-supply-chain-assurance/slsa/generate-slsa-with-github-actions).

### Usage Example

```yaml
- name: SLSA Provenance Generation
  uses: harness/github-actions/slsa-generation@1.1.0
  with:
    HARNESS_ACCOUNT_URL: https://myaccount.harness.io
    HARNESS_ACCOUNT_ID: my_account_id_9YpRharzPs
    HARNESS_ORG_ID: my_org_id_default
    HARNESS_PROJECT_ID: example_project_id
    HARNESS_API_KEY: ${{ secrets.API_KEY_SAVED_AS_GH_SECRET }}
    VAULT_ADDR: ${{ secrets.VAULT_URL }}
    TARGET: example_image:latest
    ATTEST: true
    KMS_KEY: path/to/your/key
```

### Configuration

Make sure to include the required configurations from the [Requirements](#requirements) section in your workflow. Below are the specific configurations for the `harness/github-actions/slsa-generation` sub-action.

| **Key**           | **Value Example**         | **Description**                                                                      | **Required** |
|-------------------|---------------------------|--------------------------------------------------------------------------------------|-------------|
| `TARGET`          | `example_image:latest`    | The target artifact (Docker image) for generating SLSA provenance.                  | Yes         |
| `ATTEST`          | `true` or `false`           | Boolean flag to determine if attestation is required.                               | No          |
| `KMS_KEY`         | `path/to/your/key`        | Path to the Private key used for signing the attestation.                                   | No          |


## SLSA Provenance Verification
> harness/github-actions/slsa-verification@1.1.0

The sub-action `harness/github-actions/slsa-verification` verifies the SLSA provenance attestation by pulling the `.att` file from the configured container registry. It uses the public key from the key pair that was used for signing the attestation to perform the verification. For more details, refer to [Harness documentation](https://developer.harness.io/docs/software-supply-chain-assurance/slsa/verify-slsa-with-github-actions).

### Usage Example

```yaml
- name: SLSA Verification
  uses: harness/github-actions/slsa-verification@1.1.0
  with:
    HARNESS_ACCOUNT_URL: https://myaccount.harness.io
    HARNESS_ACCOUNT_ID: my_account_id_9YpRharzPs
    HARNESS_ORG_ID: my_org_id_default
    HARNESS_PROJECT_ID: example_project_id
    HARNESS_API_KEY: ${{ secrets.API_KEY_SAVED_AS_GH_SECRET }}
    VAULT_ADDR: ${{ secrets.VAULT_URL }}
    TARGET: example_image:latest
    VERIFY: true
    KMS_KEY: path/to/your/key
```

### Configuration

Make sure to include the required configurations from the [Requirements](#requirements) section in your workflow. Below are the specific configurations for the `harness/github-actions/slsa-verification` sub-action.

| **Key**           | **Value Example**         | **Description**                                                                           | **Required** |
|-------------------|---------------------------|-------------------------------------------------------------------------------------------|-------------|
| `TARGET`          | `example_image:latest`    | The target artifact (Docker image) for which SLSA provenance verification is performed.  | Yes         |
| `VERIFY`          | `true` or `false`           | Boolean flag to determine if verification is required.                                    | Yes         |
| `KMS_KEY`         | `path/to/your/key`        | Path to the public key used for verifying the attestation.                                | No          |
