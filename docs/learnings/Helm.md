

# Upstream Charts

An upstream chart is a third-party, community-maintained Helm chart that serves as the base source for your own application deployment. It represents the original, authoritative blueprint published by open-source projects or official vendors (like [Bitnami](https://www.replicated.com/enterprise-helm) or the CNCF) to install a specific software on Kubernetes. [1, 2, 3, 4, 5] 

[Customizing Upstream Helm Charts with Kustomize | Testing ...](https://testingclouds.wordpress.com/2018/07/20/844/)
[Helm best practices · Codefresh | Docs](https://codefresh.io/docs/docs/ci-cd-guides/helm-best-practices/)
[What is Helm in Kubernetes? A complete guide | Glasskube](https://glasskube.dev/blog/what-is-helm-in-kubernetes/)

## Why Use an Upstream Chart?
Instead of building a complex Kubernetes manifest from scratch, developers pull a pre-made upstream chart to [save time and avoid replicating engineering effort](https://wikitech-static.wikimedia.org/wiki/Kubernetes/Upstream_Helm_charts_policy.html). For example, if you want to deploy PostgreSQL, you don't write your own templates; you leverage the upstream PostgreSQL chart. [3, 6, 7, 8] 
## How Upstream Charts Are Used

* As a Sub-chart (Dependency): You import the upstream chart into your own "umbrella" or wrapper chart by defining it under the dependencies key in your Chart.yaml file.
* With Post-Rendering / Kustomize: You use tools like [Kustomize](https://kustomize.io/) or Helm post-renderers to overlay your company's custom configurations (like specific security contexts or sidecars) onto the upstream manifests. This prevents you from needing to fork the original project.
* Via Custom Values: You simply inject your environment-specific variables by creating a custom values.yaml file that overrides the default settings of the upstream chart. [2, 9, 10, 11, 12, 13] 

## Summary of Best Practices
To avoid heavy technical debt, engineering teams are highly encouraged to [minimize changes to upstream code](https://docs-bigbang.dso.mil/latest/docs/developer/package-integration/upstream/). Modifying or forking an upstream chart creates "version lock," making it incredibly tedious to pull new bug fixes and security updates from the community later on. [2, 6, 14] 
Would you like help configuring an upstream chart as a dependency in your Chart.yaml, or are you looking to override specific settings using a custom values file? [9, 10, 13] 

[1] [https://ranchermanager.docs.rancher.com](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/helm-charts-in-rancher)
[2] [https://docs-bigbang.dso.mil](https://docs-bigbang.dso.mil/latest/docs/developer/package-integration/upstream/)
[3] [https://www.replicated.com](https://www.replicated.com/enterprise-helm)
[4] [https://s3.us-east-2.amazonaws.com](https://s3.us-east-2.amazonaws.com/d2iq.com/resources/solution-briefs/pure-upstream-open-source-kubernetes.pdf)
[5] [https://cloudsmith.com](https://cloudsmith.com/product/formats/helm-chart-repository)
[6] [https://wikitech-static.wikimedia.org](https://wikitech-static.wikimedia.org/wiki/Kubernetes/Upstream_Helm_charts_policy.html)
[7] [https://artifacthub.io](https://artifacthub.io/packages/helm/jfrog/jfrog-platform)
[8] [https://wikitech.wikimedia.org](https://wikitech.wikimedia.org/wiki/Helm/Upstream_Charts/cert-manager)
[9] [https://glasskube.dev](https://glasskube.dev/blog/what-is-helm-in-kubernetes/)
[10] [https://kodekloud.com](https://kodekloud.com/blog/chart-dependencies/)
[11] [https://testingclouds.wordpress.com](https://testingclouds.wordpress.com/2018/07/20/844/)
[12] [https://helm.sh](https://helm.sh/docs/topics/advanced/)
[13] [https://phalanx.lsst.io](https://phalanx.lsst.io/developers/add-external-chart.html)
[14] [https://infrastructure.2i2c.org](https://infrastructure.2i2c.org/howto/upgrade-cluster/sub-chart-version/)


To create an upstream chart for a new enterprise application, you build a standard, highly reusable generic Helm chart that acts as the single source of truth for downstream teams, environments, or clients.
The goal of an upstream enterprise chart is to package the application's core architecture while exposing flexible configuration hooks through a values.yaml file so it can be deployed anywhere without modification.
------------------------------
## 1. Initialize the Chart
Generate the scaffolding for a clean, compliant Helm layout using the command line:

helm create my-enterprise-app

This command creates a folder named my-enterprise-app containing a standard directory structure:

* Chart.yaml: The metadata blueprint.
* values.yaml: The default configuration variables.
* templates/: The Kubernetes manifest templates (Deployments, Services, Ingresses).

------------------------------
## 2. Configure Enterprise Metadata
Edit the Chart.yaml file. Because this is an upstream master chart, you must version it semantically and explicitly define its type:

apiVersion: v2name: my-enterprise-appdescription: The authoritative upstream Helm chart for the Enterprise Apptype: applicationversion: 1.0.0      # The Helm chart version (increment when templates change)appVersion: "2.4.1"  # The actual application container image versionhome: https://example.commaintainers:
  - name: Platform Engineering Team
    email: platform-infra@example.com

------------------------------
## 3. Parameterize Core Templates
To make the chart reusable as an upstream source, you must replace hardcoded values in the templates/ folder with Go template actions referencing the .Values object.
## Example: templates/deployment.yaml

apiVersion: apps/v1kind: Deploymentmetadata:
  name: {{ include "my-enterprise-app.fullname" . }}
  labels:
    {{- include "my-enterprise-app.labels" . | nindent 4 }}spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-enterprise-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-enterprise-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}

------------------------------
## 4. Expose Hooks in values.yaml
Provide safe, production-ready defaults in your values.yaml file, ensuring that downstream teams can scale or configure features easily.

# Default values for upstream enterprise applicationreplicaCount: 3
image:
  repository: ://example.com
  pullPolicy: IfNotPresent
  tag: "" # Overrides the appVersion in Chart.yaml if set
service:
  type: ClusterIP
  port: 8080
resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 512Mi
# Placeholder for downstream environments to inject database configurationsdatabase:
  host: "localhost"
  port: 5432
  secretName: "db-credentials"

------------------------------
## 5. Package and Publish to a Registry
For the chart to function as an "upstream" dependency for other developers or downstream environment pipelines, package it and push it to an enterprise OCI registry (like JFrog Artifactory, Harbor, AWS ECR, or GitHub Packages).

# 1. Package the chart into a .tgz archive
helm package ./my-enterprise-app
# 2. Login to your enterprise registry
helm registry login ://example.com -u username
# 3. Push the package to make it available as an upstream source
helm push my-enterprise-app-1.0.0.tgz oci://://example.com/helm-charts

------------------------------
## How Downstream Pipelines Consume Your New Chart
Once published, downstream teams (e.g., Staging, Production, or regional teams) pull your chart without modifying your template code. They simply reference your upstream registry in their Chart.yaml:

dependencies:
  - name: my-enterprise-app
    version: "1.0.0"
    repository: "oci://://example.com"

Would you like to set up continuous delivery (CI/CD) using GitHub Actions to publish this chart automatically, or do you need help structuring multi-tenant ingress rules for the templates?

