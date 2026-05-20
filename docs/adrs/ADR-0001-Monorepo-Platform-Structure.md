#### ADR-0001

# Monorepo Platform Structure

### _Why we organized the repository to mirror enterprise ownership boundaries_

---

|     |     |
| --- | --- |
| **ADR Number** | ADR-0001 |
| **Status** | Accepted |
| **Date** | 2026-05-19 |
| **Author** | Srinath (phoenixryzn) |
| **Project** | Enterprise Platform Engineering Lab |
| **Repository** | github.com/phoenixryzn/platform-engineering |
---
## **Context**

When starting a platform engineering project, one of the first and most consequential decisions is how to organize the repository. This shapes how hiring managers perceive the project, how ownership boundaries are communicated, and how maintainable it becomes as it grows.

Three options were considered:

- Flat structure — everything in one folder. Simple but communicates no architectural thinking.
- Tool-based structure organized by tool (helm/, terraform/, ci/). Common but conflates platform ownership with tooling.
- Ownership-based monorepo structure — mirrors how real platform teams organize work across organizational boundaries.

## **Decision**

We adopted an ownership-based monorepo structure with the following top-level directories:

|     |
| --- |
| **Repository Structure** |
|`workloads/` — Application source code. Owned by app teams. Replaceable.|
|`platform/` — Helm charts, delivery logic. Owned by platform team.|
|`infrastructure/` — Terraform, cloud provisioning. Owned by infrastructure team.|
|`ci-cd/` — GitHub Actions pipelines. Owned by platform team.|
|`environments/` — Per-environment configuration. Owned by platform team.|
|`docs/` — ADRs, runbooks, standards, learnings.|
|`scripts/` — Automation and utility scripts. |

---
## **Rationale**

### **Ownership boundaries are visible without explanation**

A new engineer joining the team should clone the repository and immediately understand who owns what. A flat structure requires tribal knowledge. An ownership-based structure encodes that knowledge in the folder hierarchy itself.

### **Workloads are explicitly separated from platform logic**

The OTel Demo application lives under workloads/. The Helm chart that deploys it lives under platform/helm/. These are different concerns with different lifecycles. The application can be replaced without touching the platform. The platform can evolve without modifying the application source.

### **Documentation is a first-class citizen**

The docs/ folder sits at the same level as platform/ and workloads/. This communicates that ADRs, runbooks, incident notes, and standards are part of the engineering deliverable — not supplementary material.

---
## **Consequences**

### **Positive**

- Any engineer cloning the repo immediately understands the operating model
- Hiring managers recognize the structure as production-grade
- Changes to one area do not require understanding of others
- The structure can be presented in interviews as a design decision with documented rationale

### **Negative**

- More initial setup than a flat structure
- Requires discipline to keep concerns in their correct directories as the project grows

|     |
| --- |
| **Principal Engineer Insight** |
| _A repository structure is not just an organizational choice. It is a communication tool. The moment a hiring manager opens your repository, they form an impression. A structure that mirrors enterprise ownership boundaries signals that you understand how platform teams actually operate — not just how to use tools._ |
---