# rabbitmq-ha-demo
Demo on RabbitMQ HA cluster deployment @ GKE 

Make sure your cluster is up and running, you can access it with kubectl and you have helm installed and tiller deployed

# tldr;
```
$ helm install stable/rabbitmq-ha
```

# customized deployment 
```
$ kubectl create -f - <<EOF                                                                                                   kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
EOF
storageclass "ssd" created
```

```
$ HELM_RELEASE_NAME=my-release
$ NAMESPACE=rabbit
$ helm install \
  --set rabbitmqUsername=admin,rabbitmqPassword=ultrasecretpassword,service.type=NodePort,persistentVolume.enabled=true,persistentVolume.storageClass=ssd,persistentVolume.size=64Gi \
  --name "$HELM_RELEASE_NAME" stable/rabbitmq-ha
```
for more parameters check the Chart repo https://github.com/kubernetes/charts/tree/master/stable/rabbitmq-ha
# open up the management UI in your browser
```
$ export NODE_IP=$( \
  kubectl get nodes --namespace "$NAMESPACE" \
    -o jsonpath="{.items[0].status.addresses[?(@.type=='ExternalIP')].address}" \
  )
$ export NODE_PORT_STATS=$( \
  kubectl get --namespace "$NAMESPACE" \
    -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}' services my-release-rabbitmq-ha \
  )
$ open http://$NODE_IP:$NODE_PORT_STATS/
```

# scaling out your cluster
The default for this helm chart are 3 RabbitMQ nodes. To scale up to 5 nodes, just upgrade the deployment.

(Warning: You have to use the existing erlang cookie secret to connect new nodes to the cluster or upgrade all of them to a new one) 
```
$ export ERLANGCOOKIE=$(\
  kubectl get secrets -n "$NAMESPACE" "$HELM_RELEASE_NAME"-rabbitmq-ha \
    -o jsonpath="{.data.rabbitmq-erlang-cookie}" | base64 --decode \
  )
$ helm upgrade \
    --set replicaCount=5,rabbitmqErlangCookie=$ERLANGCOOKIE \
    "$HELM_RELEASE_NAME" stable/rabbitmq-ha
```
# enable mirroring for all
```
$ export POD_NAME=$(\
  kubectl get pods --namespace "$NAMESPACE" -l "app=rabbitmq-ha" \
    -o jsonpath="{.items[0].metadata.name}" \
  )
$ kubectl exec $POD_NAME --namespace "$NAMESPACE" -- \
  rabbitmqctl set_policy ha-all "." \
    '{"ha-mode":"all", "ha-sync-mode":"automatic"}' --apply-to all --priority 0
Setting policy "ha-all" for pattern "." to "{"ha-mode":"all", "ha-sync-mode":"automatic"}" with priority "0" for vhost "/" ...
```
# teardown the cluster
(Warning: This will delete all the persistent data too!)
```
helm del --purge $HELM_RELEASE_NAME
kubectl delete pvc --all
```
