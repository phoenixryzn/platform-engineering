Session Summary — flagd OOMKill Investigation & Resolution
What Happened
After a successful helm install of the OTel Demo platform stack, 27 of 28 services came up healthy. The flagd pod — a feature flag service running two containers (flagd and flagd-ui as a sidecar) — was cycling between CrashLoopBackOff and OOMKilled.

Investigation Timeline
Observation 1: flagd pod showing 1/2 ready, repeatedly OOMKilling. The 2/2 state was achieved briefly then immediately lost — indicating the container started but crashed within 1 second of becoming ready.
Observation 2: Checked upstream chart defaults:
flagd container default limit:    75Mi
flagd-ui sidecar default limit:  250Mi
Both defaults were clearly insufficient for the local kind cluster environment.
Attempt 1 — Wrong property name:
Used additionalContainers to override flagd-ui resources. Schema validation rejected it:
additional properties 'additionalContainers' not allowed
Learning: Always read the upstream chart's values before guessing property names. The schema enforces structure.
Attempt 2 — Correct property name, missing required field:
Switched to sidecarContainers (correct), but omitted useDefault.env: true. Template rendering failed with:
nil pointer evaluating interface {}.env
Learning: When overriding a sidecar container, you must carry forward all required fields from the upstream definition — not just the ones you want to change. Partial overrides that drop required fields break template rendering.
Attempt 3 — Correct property, correct required fields, 512Mi limit:
yamlflagd:
  resources:
    limits:
      memory: 150Mi
  sidecarContainers:
    - name: flagd-ui
      useDefault:
        env: true
      resources:
        limits:
          memory: 512Mi
Upgrade succeeded. flagd stabilized at 2/2 Running.

Root Cause
The flagd-ui sidecar container has a startup memory spike that exceeds the upstream chart's default limit of 250Mi. The container dies within 1 second of starting — too fast to write logs — because the memory ceiling is hit during initialization. Raising the limit to 512Mi with the correct sidecar override structure resolved the issue.

Key Concepts This Incident Taught You
Schema validation is a quality gate built into Helm charts. When a chart defines a JSON schema, Helm enforces it on every install and upgrade. This prevents silent misconfigurations.
Sidecar container overrides require the full contract — you can't override just the resource limits without also carrying forward required fields like useDefault.env.
Exit code 137 always means OOMKill. Kubernetes sends SIGKILL to a container that exceeds its memory limit. No graceful shutdown, no logs — just termination.
Revision history is one of Helm's most valuable features. Every helm upgrade is a numbered, reversible checkpoint. In production this is how you roll back safely without touching Kubernetes directly.

Once flagd confirms stable, commit everything and we move to the frontend access step — the first time you'll see the running application in your browser.