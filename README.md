# How to configure and observe Istio service mesh

This repository is here to guide you through the GitHub tutorial that goes hand-in-hand with a video available on YouTube and a detailed blog post on my website. 
Together, these resources are designed to give you a complete understanding of the topic.


Here are the links to the related assets:
- YouTube Video: [What is ServiceMesh and especially what is Istio?](https://youtu.be/mOxF4cdBTd8﻿)
- Blog Post: [What is ServiceMesh and especially what is Istio?](https://isitobservable.io/observability/service-mesh/what-is-servicemesh-and-specially-what-is-istio)

Feel free to explore the materials, star the repository, and follow along at your own pace.

## K8s and Service Mesh (Istio)
<p align="center"><img src="/image/istio.png" width="40%" alt="istio Logo" /></p>

This repository showcases the usage of Istio by using GKE with the HipsterShop

## Prerequisite
The following tools need to be installed on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm
  
### 1. Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2. Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create isitobservable \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone the GitHub repo
```
git clone https://github.com/isItObservable/Episode4--Kubernetes-Istio
cd Episode4--Kubernetes-Istio
```
### 4. Deploy Istio
#### HipsterShop
```
cd hipstershop
./setup.sh
```

#### Install Istioctl
1. Download Istioctl
```
curl -L https://istio.io/downloadIstio | sh -
```
This command downloads the latest version of istio (in our case, Istio 1.10.2) compatible with our operating system.
2. Add istioctl to your PATH
```
cd istio-1.10.3
```
This directory contains samples with addons. We will refer to it later.
```
export PATH=$PWD/bin:$PATH
```
#### Install Istio
The installation of istio in your cluster will be done with the help of istioCtl
```
istionctl install  --set profile=demo -y
```
Then we want to instruct istio to automatically inject the Envoy Proxy into all the pods of our Hipster-shop application
So we will label the namespace: hipster-shoo
```
   kubectl label namespace hipster-shop istio-injection=enabled
```
To let our envoy proxy be injected, we need to restart our application.
Let's delete our workload and reapply it :
```
kubectl delete -f ./hipster-shop/k8s-manifest.yaml -n hipster-shop
kubectl apply -f ./hipster-shop/k8s-manifest.yaml -n hipster-shop 
```



##### Authorise outgoing communication to GCP
This manifest file authorizes Pods to interact with the google api (located outside of the cluster)
```
cd Episode4--Kubernetes-Istio/istio
kubectl apply -f serviceEntry_egress.yaml -n istio-system
```

#### Install Istio's addons

##### Prometheus

```
cd cd istio-1.10.3/sample/addons
kubectl apply -f prometheus.yaml
```
##### Jaeger

We also need Jaeger to collect traces in Kiali.
```
cd cd istio-1.10.3/sample/addons
kubectl apply -f jaeger.yaml
```
##### Kiali

To install Kiali, you'll need to go to the sample/addons directory :
```
cd cd istio-1.10.3/sample/addons
kubectl apply -f kiali.yaml 
```
##### grafana

To install Kiali, you'll need to go to the sample/addons directory :
```
cd cd istio-1.10.3/sample/addons
kubectl apply -f grafana.yaml 
```
#### Configure Hipster-shop

##### Delete the front-end loadbalancer service
Let's remove the default external load balancer created with hipster-shop
```
kubectl delete svc  frontend-external -n hipster-shop
```

##### Update the ingressgateway to expose ports for grafana, kiali and hipstershop
```
kubectl edit svc istio-ingressgateway -n istio-system
```
Add the following ports :
```
- name: grafana
  nodePort: 31770
  port: 8080
  protocol: TCP
  targetPort: 8182
- name: kiali
  nodePort: 31780
  port: 8181
  protocol: TCP
  targetPort: 8183
```

##### Create an Ingress gateway
To expose the Grafana, kiali and the hipstershop, deploy the manifest containing the Gateway and virtual service definition:
```
cd Episode4--Kubernetes-Istio/istio
kubectl apply -f hister-shop-gateway.yaml 
```
#### Get the external IP of the ingress gateway 
```
kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
Open your browser to look at Kiali :
http://< EXTERNAL IP >:8181

