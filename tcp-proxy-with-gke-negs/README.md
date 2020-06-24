This shows an example of using:
- Container-native load balancing with standalone NEGs
- Google Cloud TCP Proxy Load Balancing
- Kubernetes Config Connector

### Prerquisites
- Create a VPC-native cluster
- Install [Kubernetes Config Connector](https://cloud.google.com/config-connector/docs/how-to/install-upgrade-uninstall)
- Create a Deployment of your workload


### Set variables used during configuration
```
export CLUSTER_NAME="us-east4-cluster-1" # change to the name of your GKE cluster
export CLUSTER_NETWORK_TAG=$(gcloud compute instances describe $(gcloud compute instances list --filter=name~$CLUSTER_NAME --format="value(name)" | head -n 1) --format="value(tags.items)")
```

### Expose your workload deployment annotated for NEGs
```
apiVersion: v1
kind: Service
metadata:
  name: confluent-svc
  annotations:
    cloud.google.com/neg: '{"exposed_ports": {"9092":{}}}'
spec:
  type: ClusterIP
  selector:
    app: confluent
  ports:
  - port: 9092
    protocol: TCP
```

### Create VPC firewall rule
```
sed -e "s/CLUSTER_NETWORK_TAG/$CLUSTER_NETWORK_TAG/g" tcp-proxy-neg-firewall.yaml | kubectl apply -f -
```

### Create static external IP address
```
kubectl apply -f tcp-proxy-neg-ip.yaml
```

### Create health check
```
kubectl apply -f tcp-proxy-neg-health-check.yaml
```

### Create backend service
```
for neg in $(gcloud compute network-endpoint-groups list --filter=name~confluent-svc --uri); do
cat <<EOF | tee -a tcp-proxy-neg-backend-service.yaml
  - balancingMode: CONNECTION
    group:
      networkEndpointGroupRef:
              external: $neg
    capacityScaler: 1
    maxConnectionsPerEndpoint: 5
EOF
 done
```

```
kubectl apply -f tcp-proxy-neg-backend-service.yaml
```

### Create Target TCP Proxy
```
kubectl apply -f tcp-proxy-neg-target-proxy.yaml
```

### Create forwarding rule
```
kubectl apply -f tcp-proxy-neg-forwarding-rule.yaml
```
