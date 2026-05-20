ADR-0004

**GitHub Actions as CI/CD Control Plane**

_Why GitHub Actions was selected for pipeline automation and what it will govern_

|     |     |
| --- | --- |
| **ADR Number** | ADR-0004 |
| **Status** | Accepted — Implementation Planned |
| **Date** | 2026-05-19 |
| **Author** | Srinath (phoenixryzn) |
| **Current Phase** | Foundation complete. CI/CD is the next milestone. |
| **Pipeline Location** | .github/workflows/ (to be created) |

## **Context**

Every platform engineering project requires a CI/CD system to automate quality gates, validation, security scanning, and deployment. CI/CD is what transforms a manually-operated platform into a production-grade automated system.

For a portfolio targeting FAANG companies, the CI/CD layer is a primary evaluation signal. It demonstrates that you understand not just how to deploy software, but how to build the systems that make deployment safe, repeatable, and auditable.

## **Decision**

GitHub Actions was selected as the CI/CD control plane for all pipeline automation in this project.

## **Rationale**

### **Colocation with source of truth**

GitHub Actions pipelines live in the same repository as the code they govern, in .github/workflows/. The pipeline definition is version-controlled, reviewed in the same PR as the code change it validates, and auditable alongside the change history.

### **No additional infrastructure required**

Jenkins and similar CI systems require dedicated infrastructure. For a portfolio project, maintaining a CI server is overhead that distracts from the platform engineering work. GitHub Actions runs on GitHub-managed compute with zero infrastructure to provision or maintain.

### **Native GitHub integration**

GitHub Actions has first-class integration with pull request status checks, branch protection rules, environment protection gates, secrets management, and deployment tracking — enabling production-grade governance without additional tooling.

## **Planned Pipeline Architecture**

<div class="joplin-table-wrapper"><table><tbody><tr><td><p><strong>Pipeline 1: PR Validation (Every Pull Request)</strong></p></td></tr><tr><td><ul><li>helm lint — validate chart structure and templates</li><li>helm template — render manifests and fail on rendering errors</li><li>trivy — scan rendered manifests for security vulnerabilities</li><li>conftest / OPA — enforce policy-as-code rules on rendered manifests</li></ul><p></p><p>Goal: No broken chart ever reaches the main branch.</p></td></tr></tbody></table></div>

<div class="joplin-table-wrapper"><table><tbody><tr><td><p><strong>Pipeline 2: Main Branch Deploy (Every Merge to Main)</strong></p></td></tr><tr><td><ul><li>Re-run all PR validation gates</li><li>helm upgrade to dev environment</li><li>kubectl rollout status validation</li><li>Smoke test against frontend endpoint</li></ul><p></p><p>Goal: Every merge to main automatically deploys and validates in dev.</p></td></tr></tbody></table></div>

<div class="joplin-table-wrapper"><table><tbody><tr><td><p><strong>Pipeline 3: Security Scanning (Daily + Every PR)</strong></p></td></tr><tr><td><ul><li>Trivy image scanning against all container images in values files</li><li>Gitleaks secrets detection</li><li>SBOM generation with Syft</li><li>Results posted as PR comments and stored as artifacts</li></ul><p></p><p>Goal: Security visibility on every change and on a continuous schedule.</p></td></tr></tbody></table></div>

## **Branch Protection Strategy**

Main branch protection rules to be enabled once pipelines are implemented:

- Require PR review before merge
- Require all CI checks to pass before merge
- Require branches to be up to date before merge
- Restrict direct pushes to main

This transforms the repository from a personal project into something that operates like a real team repository with enforced governance.

## **Consequences**

### **Positive**

- Zero additional infrastructure to maintain
- Pipeline definitions are version-controlled alongside the code they govern
- Native GitHub integration enables branch protection and deployment tracking
- Directly transferable skill to FAANG target roles

### **Negative**

- GitHub-hosted runners have usage limits on free plans
- Pipeline implementation is a remaining milestone — project currently operates in manual mode

|     |
| --- |
| **Principal Engineer Insight** |
| _A platform without CI/CD depends entirely on human discipline to enforce its standards. CI/CD is what makes platform quality enforceable rather than aspirational. A helm lint check in a pipeline catches chart errors before they reach Kubernetes. A Trivy scan catches vulnerabilities before they reach production. The pipeline enforces the platform's quality standards automatically._ |