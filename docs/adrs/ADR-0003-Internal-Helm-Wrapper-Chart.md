ADR-0003

**Internal Helm Wrapper Chart**

_Why we wrap the upstream OTel Demo chart rather than consuming it directly_

|     |     |
| --- | --- |
| **ADR Number** | ADR-0003 |
| **Status** | Accepted |
| **Date** | 2026-05-19 |
| **Author** | Srinath (phoenixryzn) |
| **Upstream Chart** | opentelemetry-demo v0.40.8 |
| **Wrapper Location** | platform/helm/otel-platform/ |

## **Context**

The OTel Demo project publishes an official Helm chart. There are two ways to consume it:

- Direct consumption — download and modify the upstream chart directly
- Wrapper pattern — create an internal chart that declares the upstream chart as a dependency

This decision has significant long-term consequences for maintainability, upgrade safety, and organizational ownership clarity.

## **Decision**

We created an internal wrapper chart at platform/helm/otel-platform/ that declares the upstream chart as a Helm dependency. The wrapper owns no Kubernetes resource templates. All resource rendering is delegated upstream. The wrapper applies organization-specific values, environment overrides, and governance.

apiVersion: v2

name: otel-platform

description: Internal enterprise platform wrapper for OTel demo workload

type: application

version: 0.1.0

appVersion: "2.2.0"

dependencies:

\- name: opentelemetry-demo

version: 0.40.8

repository: https://open-telemetry.github.io/opentelemetry-helm-charts

## **Rationale**

### **Vendor chart modifications are unsustainable**

Modifying the upstream chart directly means every upstream release requires manually reapplying your changes. You cannot safely upgrade. Over time teams freeze on old chart versions long after security patches have been released.

### **The wrapper pattern enables controlled upgrades**

Upgrading the upstream chart is a one-line change in Chart.yaml — update the version number and run helm dependency update. Organization-specific overrides in values files remain untouched. The upgrade can be tested in dev before promoting to prod.

### **Critical implementation detail — values nesting**

Because the upstream chart is a sub-chart dependency, all values targeting it must be namespaced under the chart name:

**Correct vs Incorrect Values Nesting**

**CORRECT — nested under dependency name:**

opentelemetry-demo:

grafana:

enabled: true

**INCORRECT — at top level (upstream chart never sees this):**

grafana:

enabled: true

## **Lessons Learned From Debugging**

The flagd OOMKill incident produced three specific learnings about the wrapper pattern:

### **Schema validation is a guardrail**

The upstream chart rejected additionalContainers immediately with a schema error. This caught misconfiguration before it reached Kubernetes. Schema validation in Helm charts is a feature, not an obstacle.

### **Partial overrides must carry forward required fields**

Overriding a sidecarContainers entry without including useDefault.env: true caused a nil pointer in the template engine. The upstream definition is the contract. Partial overrides that drop required fields break the contract.

### **Sometimes the fix is removing your changes**

After multiple failed override attempts, rolling back to a clean enabled: true with no resource overrides resolved the issue. Upstream defaults exist for a reason. Override only what evidence confirms needs changing.

## **What the Wrapper Contains**

- Chart.yaml — dependency declaration and chart metadata
- values.yaml — minimal file with opentelemetry-demo: {} as namespace placeholder
- values-dev.yaml — dev environment overrides
- charts/ — downloaded upstream chart (opentelemetry-demo-0.40.8.tgz)
- Chart.lock — version receipt ensuring reproducible dependency resolution
- templates/ — intentionally empty with explanatory placeholder comment

## **Consequences**

### **Positive**

- Upstream chart upgrades are a one-line change in Chart.yaml
- Organization overrides survive upstream upgrades unchanged
- Schema validation from upstream chart catches misconfiguration early
- Mirrors how real platform teams manage vendor charts at scale

### **Negative**

- Values nesting requirement is non-obvious and a common source of confusion
- Sidecar overrides require carrying forward required fields from upstream definitions
- helm dependency update must be run after any dependency version change

|     |
| --- |
| **Principal Engineer Insight** |
| _The wrapper chart pattern is one of the clearest signals of platform engineering maturity. Any team can download a Helm chart and run helm install. A platform team builds the abstraction layer that makes that chart safe, governable, and upgradeable across an organization._ |