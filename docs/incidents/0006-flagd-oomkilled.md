# Incident 0006 — flagd OOMKilled on Initial Deploy

## Date
2026-05-19

## Observed
flagd pod showing 1/2 ready, cycling between CrashLoopBackOff 
and OOMKilled immediately after helm install. All other 27 
services healthy.

## Investigation Steps
1. Confirmed OOMKill via kubectl describe — exit code 137
2. Retrieved upstream chart defaults — flagd-ui default limit was 250Mi
3. Attempted override using additionalContainers — rejected by schema
4. Attempted override using sidecarContainers without useDefault — 
   template nil pointer error
5. Applied correct override with sidecarContainers + useDefault.env: true
   and 512Mi limit — resolved

## Root Cause
[FILL IN: Your own words describing why flagd-ui was OOMKilling]

## Resolution
Added sidecarContainers override in values-dev.yaml with:
- useDefault.env: true (required field, cannot be omitted)
- memory limit raised from 250Mi to 512Mi

## Helm Revisions
- Revision 1: Initial install — flagd OOMKilling
- Revision 2: First fix attempt — wrong property name
- Revision 3: Second fix attempt — nil pointer error  
- Revision 4: [FILL IN — did this revision fix it?]

## What I Would Do Differently
[FILL IN: Your reflection on the debugging process]

## Learning
1. Schema validation is your friend — it caught a wrong property 
   name immediately rather than letting a silent misconfiguration 
   reach the cluster.
2. Partial sidecar overrides must carry forward required upstream 
   fields. Dropping useDefault.env caused a template nil pointer.
3. A container dying in 1 second with no logs points to a startup 
   memory spike, not a gradual leak. The fix is raising the limit,
   not investigating application behavior.
4. helm upgrade creates an auditable revision history. Every fix 
   attempt is recorded and reversible.

## Actual Resolution
Attempts to override sidecarContainers introduced template errors
that compounded the original OOMKill. Rolling back to clean upstream
defaults via flagd: enabled: true without resource overrides allowed
the pod to start successfully using the chart's own defaults.

## Revised Learning
Sometimes the fix is removing your changes, not adding more.
Upstream chart defaults exist for a reason. Override only what
you can confirm needs changing through evidence, not assumption.