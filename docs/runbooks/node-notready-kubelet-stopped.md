# Runbook — Node NotReady: Kubelet Stopped Posting Status

## Symptoms
kubectl get nodes shows a worker node as NotReady.
kubectl describe node shows all conditions as Unknown with reason
"Kubelet stopped posting node status."

## Likely Causes
- kind worker container stopped or crashed
- Docker Desktop restarted without kind cluster recovering cleanly
- System sleep/hibernate interrupted the kubelet process

## Diagnosis Commands
kubectl describe node <node-name> | grep -A20 "Conditions:"
docker ps -a | grep <node-name>

## Resolution
docker restart <node-name>
# Wait 30-60 seconds, then verify:
kubectl get nodes

## Prevention
Before any Helm install, always run kubectl get nodes and confirm
all nodes are Ready. A NotReady node reduces scheduling capacity
and can cause pods to go Pending unexpectedly.