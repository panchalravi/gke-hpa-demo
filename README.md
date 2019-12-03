# gke-hpa-demo
To demonstrate application autoscale feature on GKE leveraging custom metrics

# Create GKE cluster
### Export env variables
> <code>export ZONE=asia-southeast1-a <br/>
> export REGION=asia-southeast1 <br/>
> export CLUSTER=oreilly <br/>
> export MACHINE_TYPE=n1-standard-1 <br/>
> export DISK_SIZE=10GB <br/>
> export MAX_NODES=2 <br/>
> export NUM_NODES=1 <br/>
> export PROJECT_NAME=pa-rpanchal <br/>
> gcloud config set compute/region asia-southeast1 <br/>
> gcloud components update
</code>

### Create Zonal Cluster
> <code> gcloud container clusters create $CLUSTER 
--cluster-version=latest --machine-type=$MACHINE_TYPE 
--zone=$ZONE 
--num-nodes $NUM_NODES 
--disk-size=$DISK_SIZE --max-nodes=$MAX_NODES 
--addons HorizontalPodAutoscaling,HttpLoadBalancing,KubernetesDashboard 
--no-enable-autorepair
--enable-autorepair 
--enable-autoscaling 
</code>

###  Create Regional Cluster
> <code> gcloud container clusters create $CLUSTER 
--cluster-version=latest --machine-type=$MACHINE_TYPE 
--region=$REGION 
--num-nodes $NUM_NODES 
--disk-size=$DISK_SIZE --max-nodes=$MAX_NODES 
--addons HorizontalPodAutoscaling,HttpLoadBalancing,KubernetesDashboard 
--no-enable-autorepair
--enable-autorepair 
--enable-autoscaling 
</code>

### Label atleast 2 nodes where prometheus pods would be deployed
> <code>kubectl label node <NODE_NAME> monitoring=true </code>

### Bind cluster-admin role to gcloud account
> <code> kubectl create clusterrolebinding cluster-admin-binding 
--clusterrole cluster-admin 
--user $(gcloud config get-value account)
</code>

### Delete Zonal Cluster
> <code> gcloud container clusters delete $CLUSTER --zone=$ZONE <br/>
gcloud compute disks delete $(gcloud compute disks list --format='get(name)' | tr '\n' ' ' | sed 's/ $//') -q --zone=asia-southeast1-a <br/>
gcloud compute instances stop $(gcloud compute instances list --format='get(name)' | tr '\n' ' ' | sed 's/ $//') -q --zone=asia-southeast1-a
</code>

### Delete Regional Cluster
> <code> gcloud container clusters delete $CLUSTER --region=$REGION <br/>
gcloud compute disks delete $(gcloud compute disks list --format='get(name)' | tr '\n' ' ' | sed 's/ $//') -q --region=$REGION <br/>
gcloud compute instances stop $(gcloud compute instances list --format='get(name)' | tr '\n' ' ' | sed 's/ $//') -q --region=$REGION
</code>

### Access K8s Dashboard on GKE
> <code>gcloud container clusters get-credentials <cluster name> --zone <zone> --project <project>
gcloud config config-helper --format=json | jq -r '.credential.access_token' <br/>
kubectl proxy
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
</code>

## Prometheus & Grafana setup on GKE
kubectl port-forward --namespace monitoring monitoring-prometheus-grafana-0 3000
http://localhost:3000 (Grafana UI)

### Setup Helm on client machine
> <code> wget https://get.helm.sh/helm-v2.16.1-darwin-amd64.tar.gz <br/>
tar zxvf helm-v2.16.1-darwin-amd64.tar.gz <br/>
sudo mv darwin-amd64/helm /usr/local/bin/ <br/>
rm -rf ./darwin-amd64 <br/>
rm helm-v2.16.1-darwin-amd64.tar.gz <br/>
</code>

### Setup Tiller
> <code> kubectl -n kube-system create serviceaccount tiller <br/>
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller <br/>
helm init --upgrade --service-account tiller <br/>
</code>

### Setup Prometheus Operator 
This sets up prometheus metrics collector and also ServiceMonitor/PodMonitor resources enabling custom metrics collection based on service/pod labels.
> <code> kubectl create ns monitoring <br/>
helm install --name prometheus-operator stable/prometheus-operator --namespace=monitoring --set nodeSelector.monitoring=true <br/>
#--version 6.11.0 <br/>
#helm install --name prometheus-operator stable/prometheus-operator --namespace=monitoring --set prometheusOperator.createCustomResource=false --version 6.11.0 <br/>
#helm install --name monitoring -n monitoring stable/prometheus-operator <br/>
</code>

### Allow access to 'prometheus-operator-grafana' service from external client
> <code>kubectl edit -n monitoring prometheus-operator-grafana <br/>
</code>
<ul>
    <li>Change value for type attribue from 'ClusterIP' to 'LoadBalancer'
    <li>Save the changes
</ul>

### Setup Prometheus Adapter 
This sets up metrics API server to expose custom metrics collected by prometheus, HPA can then be configured to configure auto-scaling based on custom metrics exposed through this API server.

> <code> helm install --name pa --set prometheus.url=http://prometheus-operator-prometheus.monitoring.svc.cluster.local,prometheus.port=9090 --set-string nodeSelector.monitoring=true stable/prometheus-adapter --namespace monitoring
</code>

Verify if custom metrics are available through below API endpoint
> <code> kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 </code>


## Use Case#1: Deploy CPU hungry app
> <code> git clone https://gitlab.com/wuestkamp/kubernetes-scale-that-app.git <br/>
kustomize build i | kubectl apply -f - <br/>
watch “kubectl top node && kubectl top pod” <br/>
curl http://35.246.194.4:5000 <br/>
</code>

### Hit App multiple times
> <code> curl http://35.246.194.4:5000 </code>

Edit the deployment using “kubectl edit deployments app” to define cpu limits
Hit again: curl http://35.246.194.4:5000 & see the limits are in place and working

### Create HPA
> <code> kubectl autoscale deployment app --cpu-percent=50 --min=2 --max=3 <br/>
kubectl get hpa
</code>

This should more or less maintain an average cpu usage across all pods of 50%. 
if we hit again: curl http://35.246.194.4:5000, we see we see a new pod being created

If we stress more pods, the deployment’s replicas would be increased further.
But a HPA is no good if it still allows to exceed your node pools limits. You need to calculate your resources by setting the request/limits for your containers and the number of max replicas of the HPA. Else we’ll get un-schedulable pods:

Enable nodes auto-scaling at the time of GKE cluster creation (enable-autoscaling)
Incoming WebHook App on Slack
curl -X POST --data-urlencode "payload={\"channel\": \"#alerts\", \"username\": \"webhookbot\", \"text\": \"This is posted to #alerts and comes from a bot named webhookbot.\", \"icon_emoji\": \":ghost:\"}" https://hooks.slack.com/services/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

## Use Case#2: Use custom-metrics (HTTP request rate) to auto-scale app
> <code> git clone https://gitlab.com/wuestkamp/kubernetes-scale-that-app.git <br/>
#Deploy sample application to default namespace <br/>
kubectl apply -f hpa-sim.yaml <br/> <br/>
#Configure prometheus-operator ServiceMonitor resource to scrape metrics based on your Service definition <br/
kubectl apply -f hpa-servicemonitor.yaml <br/><br/>
#Configure auto-scaling rules based on the custom metrics <br/>
kubectl apply -f hpa-autoscale.yaml <br/><br/>
#Generate load using hey(https://github.com/rakyll/hey)<br/>
hey -z 5m -c 100 -m GET http://localhost:8001/api/v1/namespaces/default/services/hpa-sim:80/proxy/service\?cost\=0.5
</code>

## Note
If you are setting up tiller on local k8s cluster, then you may have to install 'socat' on worker nodes for tiller to work. Refer below instructions to setup 'socat'.
### Setup socat on worker nodes (on local k8s cluster)
> <code> sudo apt-get update -y <br/>
sudo apt-get install -y socat
</code>