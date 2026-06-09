ADR-0010

**Terraform vs Alternatives**

_How to choose IaC tool and reason out why it is preferred over the others_

|     |     |
| --- | --- |
| **ADR Number** | ADR-0010 |
| **Status** | Accepted |
| **Date** | 2026-06-09 |
| **Author** | Srinath (phoenixryzn) |
| **Environments** | dev (planned), stage (planned), prod (planned) |

## **Context**

Every environment has setup made manually through the console by clicking on the configuration. This is required in Dev, staging and production environments having varied requirements.

Manually configuring the environment is always introducing errors while in process and there is always the clean-up process which is combersome. There are chances of orphaned resources which may not be deleted if not tracked. This creating a headache in maintenance and cost in running infrastructure in this approach.

## **Decision**

**Decision:** Terraform with S3 remote backend and DynamoDB state locking is chosen as the IaC tool for all environments. 



## **Alternatives considered**

| Tool | Reason for not considering |
|:---:|:--- |
| **CloudFormation** | AWS-only, no cross-cloud portability. Drift detection 
                  is a manual pull operation vs Terraform's automatic 
                  pre-apply diff. Stack deletion can leave orphaned 
                  resources when not properly configured. |
| **eksctl**       | No state model. Fire-and-forget API calls — orphans 
                  load balancers and security groups on delete. 
                  EKS-only, zero portability to other clouds. |
| **Pulumi**         | Requires programming language knowledge (Python, 
                  TypeScript) to author and maintain infrastructure. 
                  Adds a language dependency on top of the infra skill. |
---

## **Rationale**

Terraform is chosen as an Infrastructure as Code for the setup of the environments like dev, stage and prod. This means the Infrastructure as Code would help with the following:

* Version control of the code used to create the repository by the maintenance of the same in GitHub repository.
* Automation of creating and maintenance of the environment using Terraform and any CI/CD tool.
* Storing and maintaining the state of the infrastructure created in a backend storage such as S3 Storage + DynamoDB Database (locking the table).
* Cloud agnostic. This can help port the infrastructure requirement to other cloud providers with minimal changes.
* Easy clean-up. The infrastructure can be easily removed as compared to manual clean-up of infrastructure.

Terraform creates a `terraform.tfstate` file which would need to be stored in S3 Storage + DynamoDB table backend. This is due to the fact that `terraform.tfstate` file contains sensitive informations like secrets, passwords and access tokens which makes it a security risk if maintained in any repository (GitHub, Gitlab, GitBucket, etc.). We would need a storage which would also having state locking capabilities. This is to ensure that at a given instance, only one event is actively working on terraform.tfstate and avoid corrupting the file. 

To create this backend, we would manually create one S3 Storage and DynamoDB which would be used across the team for storing their `terraform.tfstate` and referring to the same. This is because we wouldn't need to track the drift in changes on the backend creation as they are meant to be available throughout the organization. These are considered `meta-infrastructure`.

Even in a solo project, a CI/CD pipeline trigger and a local terraform apply running simultaneously would corrupt state without the DynamoDB lock.

## **Consequences**

### **Positive**

- Cloud Agnostic: The infrastructure as code can be replicated to other clouds with minimal changes.
- State maintained: Terraform has state file which can be used to identify any drifts.
- Clean-up: Terraform helps with clean-up leaving no orphaned resource while deleting all resources.

### **Negative**

- Loss of `terraform.tfstate` can lead to loss of ability of knowing the state of the deployment. 
- Learning curve: HCL syntax and Terraform state model require upfront investment before the first successful apply
- Bootstrap exception: S3 backend and DynamoDB lock table must be created manually via AWS CLI — Terraform cannot manage its own backend without a circular dependency

|     |
| --- |
| **Principal Engineer Insight** |
| _Infrastructure management is a crucial part of the platform projects. Maintaining consistent environments creates the backbone of all the teams to function as a unit and deliver the product. The use of Terraform describes the maintenance of infrastructure as code which extends to the aspects of resilience, reusability, audit and drift detection. This also helps in the maintenance of infrastructure lifecycle and is cloud agnostic, making it easy to use across all cloud providers._|