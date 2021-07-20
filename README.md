# Is it Observable?
<p align="center"><img src="/image/logo.png" width="40%" alt="Prometheus Logo" /></p>

## K8s and Service Mesh (Istio)
<p align="center"><img src="/image/istio.png" width="40%" alt="istio Logo" /></p>
Repository containing the files for the Episode 4 of Is it Observable : K8s and Service Mesh ( Istio)


This repository showcase the usage of the Istio  by using GKE with :
- the HipsterShop


## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud containr clusters create isitobservable \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone Github repo
```
git clone https://github.com/isItObservable/Episode4--Kubernetes-Istio
cd Episode4--Kubernetes-Istio
```
### 4. Deploy Prometheus
#### HipsterShop
```
cd hipstershop
./setup.sh
```
#### Prometheus ( already done during Episde 1)
```
helm install prometheus stable/prometheus-operator
```
#### Expose Grafana
```
kubectl get svc
kubectl edit svc prometheus-grafana
```
change to type NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  labels:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 7.0.3
    helm.sh/chart: grafana-5.3.0
  name: prometheus-grafana
  namespace: default
  resourceVersion: "89873265"
  selfLink: /api/v1/namespaces/default/services/prometheus-grafana
spec:
  clusterIP: IPADRESSS
  externalTrafficPolicy: Cluster
  ports:
  - name: service
    nodePort: 30806
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
Deploy the ingress by making sure to replace the service name of your grafan
```
cd ..\grafana
kubectl apply -f ingress.yaml
```
Get the login user and password of Grafana
* For the password :
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
* For the login user:
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-user}" | base64 --decode
```
Get the ip adress of your Grafana
```
kubectl get ingress grafana-ingress -ojson | jq  '.status.loadBalancer.ingress[].ip'
```
#### Install Istioctl
1. Download Istioctl
```
curl -L https://istio.io/downloadIstio | sh -
```
This command download the latest version of istio ( in our case iostio 1.10.2) compatible with our operating system.
2. Add istioctl to you PATH
```
cd istio-1.10.2
```
this directory contains samples with addons . We will refer to it later.
```
export PATH=$PWD/bin:$PATH
```
#### Install Istio
The installation of istio in your cluster will be done with the help of istioCtl
```
istionctl install  --set profile=demo -y
```
Then we want to instruct istio to automatically inject the envoy Proxy to all the pods of our Hipster-shop application
so we will label the namesapce : hipster-shoo
```
   kubectl label namespace hipster-shop istio-injection=enabled
```
To let our envoy proxy be injected we need to restart our application.
Let's delete our workload and re-apply it :
```
kubectl delete -f ./hipster-shop/k8s-manifest.yaml -n hipster-shop
kubectl apply -f ./hipster-shop/k8s-manifest.yaml -n hipster-shop 
```

#### Configure Hipster-shop

##### Create an egress gateway
To expose the Hipster-shop out of our cluster, let's deploy a virtual service and a Gateway :
```
cd Episode4--Kubernetes-Istio/istio
kubectl apply -f hister-shop-gateway.yaml -n hipster-shop
kubectl apply -f virtualservice_frontend.yaml -n hipster-shop
```


##### Authorise outgoing communication to GCP
This manifest file authorize Pods to interact with the google api (located outside of the cluster)
```
cd Episode4--Kubernetes-Istio/istio
kubectl apply -f serviceEntry_egress.yaml -n istio-system
```

#### Install Istio's addons

##### Prometheus

To scrape Istiod stats, the following example job can be added to scrape its http-monitoring port it will be required to update the prometheus config map by adding the following job
```
- job_name: 'istiod'
  kubernetes_sd_configs:
    - role: endpoints
      namespaces:
      names:
        - istio-system
          relabel_configs:
    - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      action: keep
      regex: istiod;http-monitoring
```
To scrape Envoy stats, including sidecar proxies and gateway proxies, the following job can be added to scrape ports that end with -envoy-prom:
```
- job_name: 'envoy-stats'
  metrics_path: /stats/prometheus
  kubernetes_sd_configs:
    - role: pod

  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_container_port_name]
      action: keep
      regex: '.*-envoy-prom'
```
##### Kiali

To install Kiali you will need to go to the sample/addons directory :
```
cd istio-1.10.2/sample/addons
kubectl apply -f kiali.yaml 
```


