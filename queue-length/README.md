# Queue Length Demo
This demo will use the length of an AWS SQS Queue to autoscale a Deployment and Node Group. 

This demo is meant to simulate a user who may have a work/job queue they need completed and a queue processor. We will deploy pods that handle 1 message every 30 seconds, to simulate the work being done. The autoscalers are configured to create 1 nodes per 4 pods, simulating how many jobs one node can handle concurrently.

## Environment
Create a queue where we will enqueue work.

```bash
QUEUE_NAME=$USER-karpenter-demo-queue 
QUEUE_URL=$(aws sqs create-queue --queue-name $QUEUE_NAME --output json | jq -r '.QueueUrl') 
```

## Apply YAML
Download the manifest containing all the necessary Karpenter resources, then inject the NodeGroup and Queue you created into the manifest and apply it to the cluster.

```bash
wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/queue-length/manifest.yaml

QUEUE_NAME=$QUEUE_NAME \
QUEUE_URL=$QUEUE_URL \
QUEUE_ARN=arn:aws:sqs:$REGION:$AWS_ACCOUNT_ID:$QUEUE_NAME \
NODE_GROUP_ARN=$(aws eks describe-nodegroup --nodegroup-name demo --cluster-name ${CLUSTER_NAME} --output json | jq -r ".nodegroup.nodegroupArn") \
envsubst < manifest.yaml | kubectl apply -f -
```

## Grant SQSQueue permissions to the namespace's default service account
Update the existing service account in the `karpenter-queue-length-demo` namespace with IRSA.

```bash
eksctl create iamserviceaccount --cluster ${CLUSTER_NAME} \
--name default \
--namespace karpenter-queue-length-demo \
--attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/Karpenter" \
--override-existing-serviceaccounts \
--approve

# Due to a weird side effect of IRSA, we need to restart the Karpenter pod so that the newly created serviceaccount's secret is wired up.
kubectl delete pods -n karpenter -l control-plane=karpenter
kubectl get pods -n karpenter
```

## Watch Demo
Use the following commands to observe the demo in real time.

```bash
# Open in 7 separate terminals
# Observe the pods that will be simulating work being done
watch 'kubectl get pods -l app=subscriber -n karpenter-queue-length-demo'
watch 'kubectl get nodes'
# The MetricsProducer is responsible for monitoring the length of the queue. It will periodically retrieve the value and report it in its status. This value is scraped by prometheus and eventually used in the HorizontalAutoscaler.
watch -d 'kubectl get metricsproducer demo -n karpenter-queue-length-demo -ojson | jq ".status.queue"'
# The HorizontalAutoscaler is responsible for computing the desired number of replicas. In this case it will recommend 1 node replica per 4 messages in the queue.
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh capacity -n karpenter-queue-length-demo -ojson | jq ".status" | jq "del(.conditions)"'
# This HorizontalAutoscaler will recommend 1 pod replica per message in the queue
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh subscriber -n karpenter-queue-length-demo -ojson | jq ".status" | jq "del(.conditions)"'
# The ScalableNodeGroup is targeted by the HorizontalAutoscaler and works with EKS Managed Node Groups to reconcile the amount of node replicas.
watch -d 'kubectl get scalablenodegroup capacity -n karpenter-queue-length-demo -ojson | jq "del(.status.conditions)" | jq ".spec, .status"'
# This component is optional for autoscaling, but is useful for manually monitoring the demo by emitting %used for cpu, memory, and pods.
watch "kubectl get metricsproducers capacity-watcher -n karpenter-queue-length-demo -ojson | jq -r '.status.reservedCapacity'"
```

## Send messages to the Queue
Send 10 messages to the queue every 10 seconds.

```bash
while true ;
do
aws sqs send-message-batch --queue-url $QUEUE_URL --entries "$(cat << EOM
[
  {"Id": "0","MessageBody": " "},{"Id": "1","MessageBody": " "},{"Id": "2","MessageBody": " "},{"Id": "3","MessageBody": " "},
  {"Id": "4","MessageBody": " "},{"Id": "5","MessageBody": " "},{"Id": "6","MessageBody": " "},{"Id": "7","MessageBody": " "},
  {"Id": "8","MessageBody": " "},{"Id": "9","MessageBody": " "}
]
EOM
)" ;
sleep 10;
done
```

# Cleanup
This only cleans up the resources associated with the reserved-capacity demo. To clean up all resources, head to the [parent doc](..).

```bash
rm manifest.yaml

kubectl delete namespace karpenter-queue-length-demo
aws sqs delete-queue --queue-url $QUEUE_URL
# Clean up stacks
eksctl delete iamserviceaccount --cluster $CLUSTER_NAME --name default --namespace karpenter-queue-length-demo
```
