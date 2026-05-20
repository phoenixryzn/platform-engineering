ADR-0005

**Environment Values Strategy**

_How dev, staging, and production differences are managed without duplicating configuration_

|     |     |
| --- | --- |
| **ADR Number** | ADR-0005 |
| **Status** | Accepted |
| **Date** | 2026-05-19 |
| **Author** | Srinath (phoenixryzn) |
| **Environments** | dev (active), stage (planned), prod (planned) |
| **Values Files** | values.yaml, values-dev.yaml, values-stage.yaml, values-prod.yaml |

## **Context**

Every production platform manages multiple environments. Dev, staging, and production have different requirements: different resource limits, different component enablement, different replica counts, different security postures.

The naive approach — copying the entire chart for each environment — creates immediate divergence, makes cross-environment changes require multiple edits, and produces environments that gradually drift apart. It is used by engineers who have not operated platforms at scale.

## **Decision**

Environment differences are represented exclusively through values files. The chart itself is shared across all environments. Only the values change.

|     |
| --- |
| **Values File Hierarchy** |
| values.yaml — Minimal base. Namespace placeholder only.<br><br>values-dev.yaml — Dev overrides. Lightweight. Selective component enablement.<br><br>values-stage.yaml — Stage overrides. Closer to prod. (Planned)<br><br>values-prod.yaml — Production overrides. Full stack. Stricter limits. (Planned) |

Deployment command pattern:

\# Dev

helm upgrade otel-platform . -f values-dev.yaml --namespace otel-dev

\# Stage

helm upgrade otel-platform . -f values-stage.yaml --namespace otel-stage

\# Prod

helm upgrade otel-platform . -f values-prod.yaml --namespace otel-prod

## **Current Dev Configuration**

The active values-dev.yaml as of the initial deployment milestone:

opentelemetry-demo:

components:

frontend:

enabled: true

flagd:

enabled: true

grafana:

enabled: true

jaeger:

enabled: true

prometheus:

enabled: true

opensearch:

enabled: false # Disabled in dev — resource-intensive, not required for validation

## **Rationale**

### **Single source of truth for chart logic**

The chart templates, helpers, and dependency declarations live in one place. A bug fix applied to the chart automatically benefits all environments. No risk of fixing a bug in one environment's copy and forgetting another.

### **Differences are explicit and reviewable**

When a PR changes values-prod.yaml, the diff shows exactly what is different about the new production configuration. The values files are the audit trail for environment-specific decisions.

### **Environment promotion is a controlled operation**

Promoting a configuration from dev to stage means copying relevant values from values-dev.yaml to values-stage.yaml and submitting a PR. The chart is the same. Only the values change. This is a safe, auditable, reviewable operation.

## **Planned Stage and Prod Values**

### **Stage will add**

- opensearch: enabled: true
- Increased resource limits to mirror production capacity
- Additional replicas for critical services

### **Production will add**

- Full component enablement
- Production-grade resource limits based on observed dev and stage metrics
- Pod disruption budgets for high-availability services
- Horizontal pod autoscaling configuration
- Stricter security contexts

## **Consequences**

### **Positive**

- Single chart serves all environments — zero duplication
- Environment differences are explicit, diffable, and reviewable in PRs
- Promotion from dev to stage to prod is a controlled, auditable process
- Chart improvements benefit all environments simultaneously

### **Negative**

- Values files must be kept in sync as the chart evolves
- A values key renamed in an upstream chart upgrade must be updated in all environment files

|     |
| --- |
| **Principal Engineer Insight** |
| _Environment management is where most platform projects reveal their maturity level. Copying charts per environment is not engineering — it is copy-paste with a Kubernetes wrapper. The values strategy described here is how real platform teams manage environment differences at scale. The chart is the invariant. The values are the variable. Keep them separate._ |