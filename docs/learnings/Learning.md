# Enterprise Platform Engineering Lab — Complete Knowledge Document

### Preface
This document captures everything built, decided, debugged, and learned during the construction of a production-grade platform engineering project. It is written so that someone with no prior knowledge can read it, understand it deeply, and walk away thinking like a principal engineer.
The goal is not just to record what commands were run. The goal is to explain why every decision was made, what every tool does in plain language, and how real engineers think when things go wrong.

---
## Part 1 — System Design

### The Mission
Most people building portfolio projects make one critical mistake — they treat the application as the project. They deploy an app, take a screenshot, and call it done.

Real platform engineering works differently. In a real company, the application is just the tenant. The platform is the landlord. The platform team's job is to build the infrastructure, delivery pipelines, observability systems, security controls, and operational practices that any application can run on top of.

This project was built with that mindset. The application — OpenTelemetry Demo — is just a realistic workload used to exercise the platform. The real engineering asset is everything built around it.

---
## The Big Picture — How Everything Connects
Imagine a factory assembly line:
```
Developer writes code
        ↓
Code lives in GitHub (source of truth)
        ↓
GitHub Actions runs automated checks (CI/CD pipeline)
        ↓
Helm packages the application for Kubernetes
        ↓
Kubernetes runs the application across multiple machines
        ↓
OpenTelemetry Demo runs as the workload
        ↓
Observability stack watches everything
    ├── Prometheus collects metrics (numbers over time)
    ├── Jaeger collects traces (request journeys)
    └── Grafana displays everything as dashboards
```

Each layer has a specific job. Each layer is owned by a specific part of the repository. Each layer can be changed independently without breaking the others.

---
## The Repository Structure — Why It's Designed This Way
The repository is organized like a real company's platform team would organize it:

```
platform-engineering/
├── workloads/
│   └── otel-platform/
│       └── src/          ← The application source code lives here
├── platform/
│   └── helm/
│       └── otel-platform/    ← Your deployment logic lives here
├── infrastructure/           ← Future: Terraform, cloud provisioning
├── ci-cd/                    ← Future: GitHub Actions pipelines
├── environments/             ← Future: env-specific configs
├── docs/                     ← ADRs, runbooks, learnings
└── scripts/                  ← Automation scripts
```
Why this separation matters: In a real company, different teams own different folders. The application team owns `workloads/`. The platform team owns `platform/`. The infrastructure team owns `infrastructure/`. When everything is mixed together in one folder, you can't tell who owns what and changes break each other. This structure makes ownership visible just by looking at the repo.

A principal engineer reviewing this repo should be able to understand the operating model without asking anyone a single question.

---
## Part 2 — Components Used and What They Do

### Git and GitHub — The Foundation of Everything

**Plain English:** Git is like Google Docs with a complete history of every change ever made. GitHub is where that history lives online so the whole team can access it.

**What we used it for:** Every change made to the platform — chart updates, value overrides, documentation, fixes — was committed to Git with a meaningful message. This creates an audit trail. Six months from now you can see exactly what changed, when, and why.

**Principal engineer thinking:** A repo with meaningful commit messages is a repo that can be debugged at 3am. A repo with messages like "fix" or "update" is a liability.

---
### Kubernetes — The Operating System for Containers

**Plain English:** Imagine you have 28 different applications that all need to run at the same time, on multiple computers, with automatic restarts if they crash, and automatic load balancing if traffic spikes. Managing that manually would be a full time job for a large team. Kubernetes does all of that automatically.

**What we used it for:** Running the entire OpenTelemetry Demo stack — 28 microservices — across 4 nodes (computers) in a local cluster. Kubernetes decided which services ran on which nodes, restarted crashed containers, and enforced memory limits.

**Key concepts:**

**Pod** — the smallest unit in Kubernetes. One or more containers running together.  
**Deployment** — tells Kubernetes "I want 1 copy of this pod running at all times."  
**Service** — a stable address for a pod, since pods can move between nodes.  
**Namespace** — a logical boundary inside a cluster. Like a folder for your resources. We used otel-dev.  
**Node** — a physical or virtual machine in the cluster. We had 4: one control plane and three workers.  

**Principal engineer thinking:** Kubernetes doesn't run your app. It runs your app reliably. The difference is everything.

---
### Helm — The Package Manager for Kubernetes
**Plain English:** If Kubernetes is the operating system, Helm is the app store. Instead of writing 81 separate Kubernetes configuration files by hand, Helm lets you define an application as a reusable package called a chart, with variables you can customize per environment.

**What we used it for:** Creating an internal wrapper chart that sits on top of the official OpenTelemetry Demo Helm chart. Instead of modifying the vendor's chart directly, we built a company-owned layer on top that applies our settings, our environment overrides, and our governance.

**Key files in a Helm chart:**  
`Chart.yaml` — the identity card of your chart. Contains the name, description, version, and list of dependencies (other charts this one relies on).  
`values.yaml` — the default settings file. Every variable the chart uses has a default value here. You override these per environment.  
`values-dev.yaml` — dev environment overrides. Only the settings that differ from defaults for dev live here.  
`charts/` — folder where downloaded dependency charts live after `helm dependency update`.  
`Chart.lock` — a receipt recording the exact version of every dependency downloaded. Ensures everyone on the team gets the same thing.  
`templates/` — the actual Kubernetes config files, but with variables in them. Helm fills in the variables using values files before sending to Kubernetes.  

**The wrapper chart pattern:** The OTel Demo team publishes their own Helm chart. Instead of downloading and editing it directly, we created an internal chart that declares it as a dependency. Our chart is the control layer. Their chart does the actual work. When they release updates, we can adopt them on our schedule by changing one version number in Chart.yaml.
Principal engineer thinking: Modifying vendor charts directly is technical debt. You can never safely update the vendor without manually reapplying your changes. Wrapper charts solve this permanently.

---
### Helm CLI Commands — What Each One Does

`helm dependency update` Goes to the internet, downloads the dependency charts declared in `Chart.yaml`, stores them in `charts/`, and generates `Chart.lock`. Run this whenever you change a dependency version.  
`helm template` Renders your chart into raw Kubernetes YAML without deploying anything. This is how you see exactly what Kubernetes would receive. Run this before every deploy to catch surprises early.  
`helm lint` Runs static analysis on your chart structure. Catches formatting errors, invalid metadata, and template issues before they reach Kubernetes.  
`helm install` Deploys your chart to Kubernetes for the first time. Creates a named release with a revision number starting at 1.  
`helm upgrade` Updates an existing release with new changes. Increments the revision number. Every upgrade is recorded in Helm's history.  
`helm list` Shows all deployed releases in a namespace — name, revision number, status, and timestamp.  
`helm rollback` Reverts a release to a previous revision. This is how you undo a bad upgrade in production without touching Kubernetes directly.

**Principal engineer thinking:** Helm is not just a deployment tool. It is a release management system. The revision history it creates is your safety net.

---
### kubectl — The Control Tower for Kubernetes

**Plain English:** kubectl is the command line tool for talking to Kubernetes. You use it to check what's running, read logs, describe resources, and diagnose problems.  

**Key commands we used:**
`kubectl get nodes` — lists all machines in the cluster and their status.  
`kubectl get pods -n otel-dev` — lists all running applications in the otel-dev namespace with their health status.  
`kubectl describe pod <name> -n otel-dev` — gives the full life history of a pod including events, resource limits, restart reasons, and scheduling decisions.  
`kubectl logs <pod> -n otel-dev` — reads the application's own output. This is how you find out what the application itself says about why it's failing.  
`kubectl get events -n otel-dev --sort-by=.lastTimestamp` — shows everything Kubernetes noticed happening, in time order. Like a system log.  
`kubectl get svc -n otel-dev` — lists all services (network addresses) in the namespace.  
`kubectl port-forward svc/<name> 8080:8080 -n otel-dev` — creates a tunnel from your laptop's port 8080 to a service inside Kubernetes. Used for local access without setting up ingress.  

---
### OpenTelemetry Demo — The Workload

**Plain English:** A realistic fake e-commerce application built by the OpenTelemetry project to demonstrate distributed tracing, metrics, and observability. It has 28 microservices written in different programming languages — Go, Python, Java, Rust, JavaScript, and more — all communicating with each other.

**Why we chose it:** It's complex enough to be realistic, it has a built-in observability stack, and it's less overused in portfolios than Google's Online Boutique.

**What runs inside it:**

Frontend and frontend proxy — the web interface  
Product catalog, cart, checkout, payment, shipping — e-commerce services  
Kafka — message queue for async communication  
PostgreSQL and Valkey — databases  
Prometheus, Grafana, Jaeger — observability stack  
OTel Collector — the observability pipeline that collects signals from all services  
flagd — feature flag service for toggling features at runtime  
Load generator — automatically generates fake traffic so dashboards have data  

---
### The Observability Stack — Watching the Platform

**Prometheus — Metrics Collection** Collects numbers over time from every service. CPU usage, request counts, error rates, memory consumption. Think of it as the platform's vital signs monitor. It scrapes data from services on a schedule and stores it as time-series data.

**Grafana — Visualization** Connects to Prometheus and displays its data as dashboards. Graphs, gauges, heatmaps. This is the screen your SRE team watches during an incident.

**Jaeger — Distributed Tracing** When a user clicks "buy" in the frontend, that request touches 8 different services before completing. Jaeger records the entire journey — how long each service took, where it failed if it failed, and what the full path looked like. This is how you debug slow requests in a microservices architecture.

**OTel Collector — The Pipeline** All 28 services send their telemetry data (metrics, traces, logs) to the OTel Collector. The Collector processes, filters, and routes that data to the right backend — Prometheus for metrics, Jaeger for traces. It's the central nervous system of the observability stack.

---
## Part 3 — Implementation — What We Did and Why

### Phase 1 — Choosing the Right Workload
**What happened:** We initially considered Google Online Boutique as the demo application. It was rejected because it's overused in portfolios and doesn't push toward observability and distributed systems thinking.

**Decision:** Switch to OpenTelemetry Demo. It's more complex, more realistic, and directly aligned with the SRE and Platform Engineering roles being targeted.

**Principal engineer thinking:** The workload you choose signals your ambition. A hiring manager who has seen fifty Online Boutique portfolios will notice when someone chose something harder and more current.

---
### Phase 2 — Designing the Repository Structure

**What happened:** Before writing a single line of code, we designed the folder structure to mirror real organizational ownership boundaries.

**Decision:** Separate workloads/, platform/, infrastructure/, ci-cd/, environments/, docs/, and scripts/ at the top level.

**Why it matters:** When a hiring manager clones your repo, the first thing they see is the folder structure. If it looks like a tutorial project with everything dumped in one place, they've already formed an opinion. If it looks like something a real platform team would build, they lean forward.

---
### Phase 3 — Building the Wrapper Chart

**What happened:** We ran helm create otel-platform inside platform/helm/otel-platform/ to generate a chart scaffold. Then we replaced all the default scaffold content with wrapper chart content.

**Three critical changes:**
First, we updated Chart.yaml to declare the upstream OTel Demo chart as a dependency and gave the chart a meaningful description instead of the default "A Helm chart for Kubernetes."  
Second, we replaced the default values.yaml — which contained nginx configuration completely unrelated to our project — with a minimal file that simply namespaces all configuration under opentelemetry-demo:.  
Third, we cleared the templates/ folder of all default scaffold templates (Deployment, Service, Ingress, HPA) since our wrapper chart doesn't define its own Kubernetes resources. It delegates everything to the upstream chart.  

**Why clearing templates matters:** A wrapper chart that still has a default nginx Deployment template is a wrapper chart that will try to deploy nginx alongside OTel Demo. The default scaffold is a starting point, not a finished product. A principal engineer always cleans up scaffolding before committing.

---
### Phase 4 — Pulling the Dependency

**What happened:** Running helm dependency update contacted the OpenTelemetry Helm chart repository, downloaded version 0.40.8 of the opentelemetry-demo chart, stored it as a compressed file in charts/, and generated Chart.lock.

**Key learning:** The Helm chart for OTel Demo and the source code of OTel Demo are two completely different things. The source code lives in workloads/otel-platform/src/. The Helm chart comes from the internet via the dependency system. They are not the same and should never be confused.

---
### Phase 5 — Creating Environment Values

**What happened:** We created values-dev.yaml with explicit settings for the dev environment — enabling frontend, Grafana, Jaeger, and Prometheus while disabling OpenSearch (not needed in dev).

**The nesting rule:** Because our chart is a wrapper, every key that targets the upstream chart must be nested under opentelemetry-demo:. This is how Helm passes values into sub-charts. A key written at the top level of your values file will be ignored by the upstream chart.
yaml# This works — nested under the dependency name
```
opentelemetry-demo:
  grafana:
    enabled: true
```

# This does nothing — the upstream chart never sees it
```
grafana:
  enabled: true
```
**Principal engineer thinking:** The nesting rule is one of the most common mistakes in wrapper charts. Understanding why it works this way — Helm's sub-chart value scoping — is what separates someone who follows instructions from someone who understands the system.

---
### Phase 6 — Render, Lint, Install

**Render first:** Running helm template produced a single YAML file containing all 81 Kubernetes objects that would be deployed. We inspected it before touching Kubernetes. This is a mandatory safety habit — you never deploy blind.

**Lint second:** Running helm lint confirmed the chart structure was valid. One informational note about a missing icon — irrelevant. Zero errors or warnings.

**Install third:** Running helm install with --wait --timeout 10m deployed all 28 services and waited for them to reach ready state before reporting success. The --wait flag is critical because without it Helm reports success the moment it submits the manifests to Kubernetes, before a single pod has actually started.

---

## Part 4 — Issues Faced and Diagnostic Steps

Issue 1 — Empty volumeMounts and volumes in Rendered Output
What was observed: After running helm template, the rendered YAML showed bare volumeMounts: and volumes: keys with nothing under them for many components.
Initial concern: This looked like a rendering bug — like the wrapper chart was stripping volume configurations from the upstream chart.
Diagnostic approach: Instead of immediately trying to fix it, we compared our render against the upstream chart rendered directly without the wrapper:
bash# Our wrapper render
grep -c "volumeMounts:$\|volumes:$" rendered-dev.yaml
# Result: 57

# Upstream direct render  
grep -c "volumeMounts:$\|volumes:$" rendered-upstream.yaml
# Result: 60
Finding: The upstream chart produces more empty volume blocks than our wrapper does. This is an intentional upstream chart pattern — it renders the keys even when no volumes are configured.
Resolution: Non-issue. Kubernetes accepts empty volume keys and treats them as no volumes defined.
Long term prevention: Always establish a baseline comparison before assuming a rendering artifact is caused by your changes. This is a fundamental debugging discipline — isolate the variable before drawing conclusions.

Issue 2 — Node NotReady Before Install
What was observed: Before installing, kubectl get nodes showed desktop-worker with status NotReady. All conditions showed Unknown with reason Kubelet stopped posting node status.
Diagnostic approach:
bashkubectl describe node desktop-worker | grep -A20 "Conditions:"
Confirmed the kubelet process had stopped reporting at a specific timestamp. The node had plenty of capacity (8 CPU, 16GB RAM) so this was not a resource problem.
Root cause: The kind cluster worker node's kubelet process stopped — likely due to Docker Desktop restarting or the system sleeping.
Fix: Restarted the Docker container running the worker node:
bashdocker restart desktop-worker
Node recovered within 60 seconds.
Long term prevention: Written as a permanent runbook at docs/runbooks/node-notready-kubelet-stopped.md. Before any future install, always run kubectl get nodes and confirm all nodes are Ready. A NotReady node reduces scheduling capacity and causes pods to go Pending in ways that look like application failures.

Issue 3 — flagd OOMKilled After Install
What was observed: After helm install succeeded, 27 of 28 services came up healthy. The flagd pod showed 1/2 ready and was cycling between CrashLoopBackOff and OOMKilled.
What OOMKilled means: Kubernetes enforces memory limits on every container. When a container exceeds its limit, Kubernetes sends an immediate kill signal — no warning, no graceful shutdown. Exit code 137 always means OOMKill.
Diagnostic steps:
Step 1 — Identified which container was failing. The pod had two containers: flagd and flagd-ui. The 1/2 ready status meant one was healthy and one was not.
Step 2 — Checked upstream chart defaults:
flagd container default memory limit:    75Mi
flagd-ui sidecar default memory limit:  250Mi
Step 3 — Observed the container was dying within 1 second of starting with no logs. A container that produces no logs before dying is hitting OOM during initialization — a startup memory spike, not a gradual leak.
Fix attempts and what each taught us:
Attempt 1 — Wrong property name:
Used additionalContainers to override the flagd-ui sidecar. The chart's JSON schema validation immediately rejected it:
additional properties 'additionalContainers' not allowed
Learning: Helm charts with JSON schemas enforce their structure at upgrade time. This is a feature, not a bug. It prevents silent misconfigurations. Always read the upstream values before guessing property names.
Attempt 2 — Correct property, missing required field:
Switched to sidecarContainers (correct), but omitted useDefault.env: true. Template rendering failed:
nil pointer evaluating interface {}.env
Learning: When overriding a sidecar container definition, you must carry forward all required fields from the upstream definition. A partial override that drops required fields breaks the template engine.
Attempt 3 — Correct property, correct required fields:
Added useDefault.env: true with 512Mi limit. Upgrade succeeded but flagd continued OOMKilling.
Actual resolution: Rolling back to a clean flagd: enabled: true without any resource overrides allowed the pod to start successfully using upstream defaults. The repeated failed override attempts had left the pod in a broken state. A clean rollout resolved it.
The real learning: Sometimes the fix is removing your changes, not adding more. Upstream chart defaults exist for a reason and were tested by the chart authors. Override only what you can confirm needs changing through evidence, not assumption. When a pod is in a broken state from multiple failed attempts, a clean rollout often resolves cascading issues that accumulated during debugging.
Long term prevention:

Always read upstream values completely before writing overrides
Test one change at a time — multiple simultaneous changes make root cause analysis impossible
Document failed attempts as part of the incident record, not just the resolution
Recognize that a container dying in under 2 seconds with no logs is a startup spike, which may resolve itself under clean conditions


Part 5 — Where the Project Stands Now
Helm release: otel-platform deployed in namespace otel-dev, currently at Revision 4.
Stack status: 28/28 services Running.
Repository: Committed and pushed to GitHub with meaningful commit messages at every checkpoint.
Documentation created:

Incident 0005 — empty volume blocks investigation
Incident 0006 — flagd OOMKill investigation and resolution
Runbook — node NotReady kubelet stopped

Immediate next step: Access the frontend via port-forward and confirm the application is serving traffic in the browser.
Coming next:

GitHub Actions CI pipeline for lint and render validation
Trivy security scanning
Branch protection rules
Architecture Decision Records
Grafana dashboard exploration
Chaos engineering exercises