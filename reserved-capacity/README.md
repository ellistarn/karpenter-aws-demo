# Reserved Capacity Demo

## Apply YAML
Download the manifest and inject your nodegroup details into it.

```bash
wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/reserved-capacity/manifest.yaml

NODE_GROUP_ARN=$(aws eks describe-nodegroup --nodegroup-name demo --cluster-name $USER-karpenter-aws-demo --output json | jq -r ".nodegroup.nodegroupArn") \
envsubst < manifest.yaml | kubectl apply -f -
```
## Watch
Use the following commands to observe the demo in real time.

```bash
# Open in 5 separate terminals
watch 'kubectl get pods -n karpenter-reserved-capacity-demo'
watch 'kubectl get nodes'
# The MetricsProducer is responsible for calculating the %used of memory, cpu, and pods for the cluster. It will periodically calculate the value and report it in its status. This value is scraped by prometheus and eventually used in the HorizontalAutoscaler
watch -d 'kubectl get metricsproducers.autoscaling.karpenter.sh demo -n karpenter-reserved-capacity-demo -ojson | jq ".status.reservedCapacity"'
# The HorizontalAutoscaler is responsible for computing the desired number of replicas. In this case it will recommend scale up of the nodes to remain at 60% usage for memory and cpu.
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh capacity -n karpenter-reserved-capacity-demo -ojson | jq ".status" | jq "del(.conditions)"'
# The ScalableNodeGroup is targeted by the HorizontalAutoscaler and works with EKS Managed Node Groups to reconcile the amount of node replicas.
watch -d 'kubectl get scalablenodegroups.autoscaling.karpenter.sh capacity -n karpenter-reserved-capacity-demo -ojson | jq "del(.status.conditions)"| jq ".spec, .status"'
```

## Scale the Pods and Nodes
Manually change the replicas of the deployments.

```bash
wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/reserved-capacity/inflate.yaml

REPLICAS=2 envsubst < inflate.yaml | kubectl apply -f -
REPLICAS=10 envsubst < inflate.yaml | kubectl apply -f -
REPLICAS=30 envsubst < inflate.yaml | kubectl apply -f -
```

## Cleanup
This only cleans up the resources associated with the queue-length demo. To clean up all resources, head to the [parent doc](..).

```bash
rm manifest.yaml
rm inflate.yaml

kubectl delete namespace karpenter-reserved-capacity-demo
# Clean up stacks
eksctl delete iamserviceaccount --cluster $CLUSTER_NAME --name default --namespace karpenter-reserved-capacity-demo
```
