#### ADR-0002

# OpenTelemetry Demo as Workload

### _Why we chose OTel Demo over simpler alternatives and how we frame its role_

--- 

|     |     |
| --- | --- |
| **ADR Number** | ADR-0002 |
| **Status** | Accepted |
| **Date** | 2026-05-19 |
| **Author** | Srinath (phoenixryzn) |
| **Upstream Chart** | opentelemetry-demo v0.40.8 |
| **Replaces Consideration** | Google Online Boutique |
--- 
## **Context**

Every platform engineering project needs a realistic workload. The workload must be complex enough to demonstrate real operational concerns while remaining manageable for a single engineer building a portfolio.

The critical framing decision: the workload is not the project. The platform is the project. The workload is what runs on top of it.

## **Alternatives Considered**

<div class="joplin-table-wrapper"><table><tbody><tr><td><p><strong>Option 1: Google Online Boutique — Rejected</strong></p></td></tr><tr><td><p>A microservices e-commerce demo by Google. Well-documented, widely used, good Kubernetes support.</p><p></p><p><strong>Rejected because:</strong></p><ul><li>Overused in DevOps portfolios — most hiring managers have seen dozens of Boutique-based projects</li><li>Does not include a built-in observability stack</li><li>Provides less learning surface for distributed tracing and OTel concepts</li></ul></td></tr></tbody></table></div>

<div class="joplin-table-wrapper"><table><tbody><tr><td><p><strong>Option 2: OpenTelemetry Demo — Selected</strong></p></td></tr><tr><td><p>A distributed microservices app built by the OTel project to demonstrate observability. 28 services in multiple languages with Prometheus, Grafana, Jaeger, and OTel Collector built in.</p><p></p><p><strong>Selected because:</strong></p><ul><li>Less common in portfolios — signals deliberate choice over default selection</li><li>Built-in observability stack exercises real SRE skills out of the box</li><li>28 services across Go, Python, Java, Rust, JavaScript provide realistic operational complexity</li><li>Directly aligned with Platform Engineer and SRE target roles</li><li>flagd feature flag service adds operational complexity not present in simpler demos</li></ul></td></tr></tbody></table></div>

## **The Framing Principle**

|     |
| --- |
| **Platform as Landlord, Workload as Tenant** |
| **The OpenTelemetry Demo is the tenant. The platform is the landlord.**<br><br>The workload is replaceable. If a better demo application emerges, the platform should adopt it without redesigning the delivery architecture, observability stack, or CI/CD pipeline. Every platform decision is made with workload-agnosticism as a constraint. |

## **What This Workload Produced**

The complexity of OTel Demo generated real operational incidents that became the project's best documentation:

- Node kubelet failure — diagnosed and resolved, now a permanent runbook
- Empty volume block analysis — required baseline comparison methodology to rule out wrapper-introduced artifacts
- flagd OOMKill — a four-revision debugging exercise covering schema validation, nil pointer errors, and partial override constraints

A simpler workload would have produced none of these. The incidents are the portfolio, not the deployment.

## **Consequences**

### **Positive**

- Realistic complexity produces real incidents worth documenting as runbooks and postmortems
- Built-in observability demonstrates SRE skills without additional setup
- Differentiated from the majority of DevOps portfolios

### **Negative**

- Higher resource requirements — requires 3+ worker nodes with adequate RAM
- 28-service complexity means more surface area for things to go wrong
- flagd sidecar architecture introduced non-obvious debugging challenges

|     |
| --- |
| **Principal Engineer Insight** |
| _The choice of workload is itself a portfolio signal. Choosing the most common option says "I followed the tutorial." Choosing a harder option says "I sought out complexity deliberately." The OTel Demo is harder. That difficulty created the incidents that became this project's most compelling content._ |