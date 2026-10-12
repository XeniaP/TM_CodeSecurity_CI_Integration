# TrendAI_CodeSecurity_CI_Integration

CI/CD pipeline examples for integrating **Trend Micro Vision One Code Security** — the **Artifact Scanner (TMAS)** and the **Infrastructure as Code (IaC) Scan** — into your build and deployment pipelines.

These pipelines scan container images and IaC templates for vulnerabilities, malware and secrets, evaluate the **Code Security policies** configured in Vision One, and act as a **security gate**: they approve the build when everything is clean, or block it when a policy violation is detected.

## What's inside

- **Azure DevOps — Container Security (SCA)**: scans container images with the TMAS CLI and enforces the Code Security policy gate.
- **Azure DevOps — Infrastructure as Code (IaC)**: scans Terraform and CloudFormation templates with the Vision One Template Scanner.
- **Jenkins**: declarative pipeline that runs TMAS against a container image before deployment and blocks the build on policy violation.
- **AWS CodeBuild**: official buildspec that scans a locally built Docker image with TMAS before pushing it to Amazon ECR.

## Repository structure

| Path | Platform | Description |
|------|----------|-------------|
| [AzureDevOps/SCA/](AzureDevOps/SCA/) | Azure DevOps | Container image scan (TMAS) pipeline with security gate |
| [AzureDevOps/IaC/](AzureDevOps/IaC/) | Azure DevOps | Infrastructure as Code (Terraform / CloudFormation) scan pipeline |
| [Jenkins/](Jenkins/) | Jenkins | TMAS scan pipeline with security gate |
| [AWS_Commit/buildspec.yml](AWS_Commit/buildspec.yml) | AWS CodeBuild | Buildspec that scans an image with TMAS before pushing to ECR |

## Getting started

### Prerequisites

- A **Trend Micro Vision One** account.
- A **Vision One API key** created in the [Vision One console](https://cloudone.trendmicro.com/) (see the [API Key Creation Guide](https://docs.trendmicro.com/en-us/documentation/article/trend-vision-one-platform-api-keys)). For best practices, use a custom role with only the minimum required permissions.
- The **Code Security** add-on enabled in your Vision One deployment (required for policy evaluation with `--evaluatePolicy`).

### Azure DevOps

Two options are provided, depending on what you want to scan:

#### Container images (SCA)

1. Create a secret pipeline variable named `TMAS_API_KEY` with your Vision One API key.
2. Copy [`AzureDevOps/SCA/azure-pipelines.yml`](AzureDevOps/SCA/azure-pipelines.yml) into your repository, or use the snippet documented in the [SCA README](AzureDevOps/SCA/README.md).
3. Provide the `IMAGE` and `TMAS_REGION` parameters when running the pipeline.

The pipeline installs the TMAS CLI, scans the image with the Vulnerability, Malware and Secrets modules (`-VMS`), evaluates your Code Security policies, publishes the results as a pipeline artifact, and finishes with a `SECURITY GATE PASSED` / `SECURITY GATE BLOCKED` verdict.

#### Infrastructure as Code (IaC)

1. Create a secret pipeline variable named `TM_API_KEY` with your Vision One API key.
2. Use the [`AzureDevOps/IaC/tm_iac_pipeline.yaml`](AzureDevOps/IaC/tm_iac_pipeline.yaml) pipeline (full example) or the snippet documented in the [IaC README](AzureDevOps/IaC/README.md).

The pipeline auto-detects supported Terraform and CloudFormation template files and scans them using the `ghcr.io/xeniap/tm_scanner_ci` scanner image. For Terraform, a plan file in JSON format is required — generate it with `terraform show -json`.

### Jenkins

1. Add a Jenkins credential of type **Secret text** with ID `tmas-api-key` holding your Vision One API key.
2. Use the [`Jenkins/pipeline`](Jenkins/pipeline) file as your pipeline definition (see the [Jenkins README](Jenkins/README.md) for full details).

The pipeline downloads the TMAS CLI on the agent, scans the image with `-VMS --evaluatePolicy`, and:

- **exit code `0`** → build continues (gate passed)
- **exit code `2`** → build fails with *SECURITY GATE BLOCKED: Policy violation*
- **any other code** → build fails with *TMAS scan failed*

Both `tmas-results.json` and `tmas-scan.log` are archived as build artifacts on every run.

### AWS CodeBuild

The [buildspec](AWS_Commit/buildspec.yml) walks through the full workflow:

1. Authenticates to Amazon ECR and builds / tags the Docker image.
2. Retrieves the TMAS API key from **AWS Systems Manager Parameter Store**.
3. Downloads the TMAS CLI, scans the locally built image (`--malwareScan`) and saves the findings to `findings.json`.
4. Pushes the image to ECR and publishes `findings.json` and `imagedefinitions.json` as build artifacts.

> **Note:** Define the `IMAGE_NAME`, `IMAGE_TAG`, `TMAS_API_KEY_PARAMETER_NAME`, `AWS_REGION` and `AWS_ACCOUNT_ID` variables in your CodeBuild project (the API key parameter must be stored with `--with-decryption`).

## How the security gate works

TMAS generates an **SBOM** from the scanned artifact, sends it to **Trend Vision One** for processing, and returns a vulnerability / malware / secrets report together with the evaluated policy decision. The pipeline then decides the build outcome:

| TMAS exit code | Meaning | Pipeline result |
|----------------|---------|-----------------|
| `0` | Scan completed, no policy violations | ✅ Approved — security gate passed |
| `2` | A Code Security policy was violated | 🛑 Blocked — security gate failed |
| other | Scan or infrastructure error | 🛑 Blocked — scan failed |

## Official documentation

- [Integrating TMAS into CI/CD pipelines (Trend Vision One)](https://docs.trendmicro.com/en-us/documentation/article/trend-vision-one-integrating-tmas-ci-cd-pipeline)
- [Vision One Artifact Scanner (TMAS) — overview](https://docs.trendmicro.com/en-us/documentation/article/trend-vision-one-artifact-scanner-tmas)
- [Vision One Infrastructure as Code — Template Scanner](https://docs.trendmicro.com/en-us/documentation/article/trend-vision-one-__template-scanner-2)
- [Vision One API Keys Guide](https://docs.trendmicro.com/en-us/documentation/article/trend-vision-one-platform-api-keys)

## Contributing

Contributions are welcome! 🎉

If you encounter any issues, bugs, or have suggestions for improvements, please feel free to [open an issue](https://github.com/xeniap/TM_CodeSecurity_CI_Integration/issues).

To contribute code:

1. Fork the repository.
2. Create a feature branch (`git checkout -b feature/your-improvement`).
3. Commit your changes and push them to your fork.
4. Open a pull request describing the change and how to test it.

We appreciate your feedback and collaboration to make this integration better for everyone.