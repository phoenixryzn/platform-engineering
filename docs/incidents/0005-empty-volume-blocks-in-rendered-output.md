# Incident 0005 — Empty volumeMounts and volumes blocks in rendered output

## Observed
helm template rendering showed bare `volumeMounts:` and `volumes:` keys
with no content under them across ~57 component pods.

## Investigation
Compared against upstream chart rendered directly without wrapper:
- Wrapper render: 57 empty blocks
- Upstream direct render: 60 empty blocks

## Root Cause
Upstream opentelemetry-demo chart version 0.40.8 intentionally renders
volume keys even when no volumes are configured for a component.
This is a chart templating pattern, not a misconfiguration.

## Resolution
Non-issue. Kubernetes accepts empty volume keys and treats them as
no volumes defined. No action required.

## Learning
Always compare wrapper render against upstream render before assuming
a rendering artifact is caused by the wrapper. Baseline comparison is
a core debugging discipline.

## Debugging Steps

This is not a bug in your wrapper chart. Let's verify whether Kubernetes actually rejects this by checking if the upstream chart itself renders this way:
```bash
    grep -c "volumeMounts:$\|volumes:$" rendered-dev.yaml
```
Then check what the upstream chart's own default render looks like by pulling it directly:
```bash
    helm template test-upstream oci://ghcr.io/open-telemetry/opentelemetry-helm-charts/opentelemetry-demo --version 0.40.8 > rendered-upstream.yaml 2>&1

    grep -c "volumeMounts:$\|volumes:$" rendered-upstream.yaml
```
If the upstream chart renders the same empty blocks without your wrapper involved, then this is an upstream chart quirk — not something you introduced. Kubernetes actually accepts empty volumeMounts: and volumes: keys in most versions and simply treats them as no volumes defined.
If the count is zero in the upstream render but non-zero in yours, then something in your values-dev.yaml is stripping values that should be there.
