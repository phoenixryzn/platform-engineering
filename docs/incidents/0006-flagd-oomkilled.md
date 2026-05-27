# Incident 0006 — flagd OOMKilled on Initial Deploy

## Date
2026-05-19

## Observed
flagd pod showing 1/2 ready, cycling between CrashLoopBackOff
and OOMKilled immediately after helm install. All other 27
services healthy.

## Investigation Steps
1. Confirmed OOMKill via kubectl describe — exit code 137
2. Identified two containers in the flagd pod: flagd (main) 
   and flagd-ui (sidecar)
3. Retrieved upstream chart defaults — flagd-ui default memory 
   limit: 250Mi
4. Observed container dying in under 1 second with no logs — 
   confirmed startup memory spike, not a gradual leak
5. Attempted resource override using additionalContainers — 
   rejected by upstream chart JSON schema validation
6. Attempted override using sidecarContainers without useDefault 
   field — template nil pointer error
7. Applied correct override with sidecarContainers + 
   useDefault.env: true and 512Mi limit — upgrade succeeded 
   but OOMKill continued
8. Reverted all overrides to upstream defaults — pod stabilised

## Root Cause
The flagd-ui sidecar container experiences a memory spike during
initialisation that exceeds the upstream chart default of 250Mi.
The container dies in under 1 second before producing any logs.
Multiple failed override attempts compounded the issue by 
introducing template rendering errors alongside the original 
OOMKill.

## Resolution
Reverted values-dev.yaml to upstream defaults with no resource
overrides for flagd:

  flagd:
    enabled: true

The upstream chart defaults (250Mi for flagd-ui) proved sufficient
once the overriding attempts were removed and the pod received a
clean rollout.

## Helm Revisions
- Revision 1: Initial install — flagd OOMKilling on startup
- Revision 2: Override attempt using additionalContainers — 
  rejected by chart schema validation
- Revision 3: Override attempt using sidecarContainers without 
  useDefault.env — nil pointer template error
- Revision 4: Clean rollout with upstream defaults restored — 
  flagd stabilised at 2/2 Running

## What I Would Do Differently
1. Read the upstream chart values completely before writing any 
   override — the sidecarContainers structure and required fields 
   were documented in the chart, not guessed
2. Make one change at a time — multiple simultaneous override 
   attempts made root cause analysis harder
3. Establish a baseline first — render the upstream chart directly 
   without any wrapper overrides to confirm whether the issue 
   exists in the base chart before assuming the wrapper introduced it
4. Check if the issue reproduces on a fresh install before applying 
   fixes — the pod may have stabilised on its own given a clean 
   cluster state

## Learning
1. Schema validation is a guardrail, not an obstacle — it caught 
   a wrong property name before it reached Kubernetes
2. Partial sidecar overrides must carry forward all required fields 
   from the upstream definition — omitting useDefault.env caused 
   a nil pointer in the template engine
3. A container dying in under 1 second with no logs indicates a 
   startup memory spike — the fix is a clean environment, not 
   necessarily a higher limit
4. helm upgrade creates an auditable revision history — every 
   attempt is recorded and reversible, which aided diagnosis
5. Sometimes the fix is removing your changes — upstream defaults 
   exist for a reason and were validated by the chart authors

## Prevention
Before overriding any sidecar container resource in a wrapper 
chart:
- Read the full upstream values definition for that component
- Test one override at a time in isolation
- Confirm the issue exists without your overrides before adding them