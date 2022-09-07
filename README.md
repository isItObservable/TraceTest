# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Episode : TraceTest
This repository contains the files utilized during the tutorial presented in the dedicated IsItObservable episode related to TraceTest.
<p align="center"><img src="/image/tracetest.png" width="40%" alt="TraceTest Logo" /></p>

What you will learn
* How to deploy TraceTest
* how to configure your OpenTelemetry Pipeline
* how to Build a traceTest test case

This repository showcase the usage of the OpenTelemtry Collector with :
* the HipsterShop
* K6 ( to generate load in the background)
* The OpenTelemetry Operator
* Nginx ingress controller
* Prometheus
* Tempo
* TraceTest

## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm


## Deployment Steps in GCP

You will first need a Kubernetes cluster with 2 Nodes.
You can either deploy on Minikube or K3s or follow the instructions to create GKE cluster:
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
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```

### 3.Clone the Github Repository
```
git clone https://github.com/isItObservable/TraceTest
cd TraceTest
```
### 4.Deploy Nginx Ingress Controller 
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
#### 1. get the ip adress of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* hipstershop
```
IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```
#### 2. Update the manifest files

update the following files to update the ingress definitions :
```
sed -i "s,IP_TO_REPLACE,$IP," kubernetes-manifests/k8s-manifest.yaml
sed -i "s,IP_TO_REPLACE,$IP," grafana/ingress.yaml
```


### 3.Prometheus

#### 1. Deploy Prometheus without Grafana
 ```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack  
```

#### 2. Enable remote Writer
To be able to send the Collector metrics to Prometheus , we need to enable the remote writer feature.
To enable this feature we will need to edit the CRD containing all the settings of promethes: prometehus

To get the Prometheus object named use by prometheus we need to run the following command:
```
kubectl get Prometheus
```
here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.32.1   1          22h
```
We will need to add an extra property in the configuration object :
```
enableFeatures:
- remote-write-receiver
```
so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```
After the update your Prometheus object should look  like :
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  generation: 2
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 30.0.1
    chart: kube-prometheus-stack-30.0.1
    heritage: Helm
    release: prometheus
  name: prometheus-kube-prometheus-prometheus
  namespace: default
spec:
  alerting:
  alertmanagers:
  - apiVersion: v2
    name: prometheus-kube-prometheus-alertmanager
    namespace: default
    pathPrefix: /
    port: http-web
  enableAdminAPI: false
  enableFeatures:
  - remote-write-receiver
  externalUrl: http://prometheus-kube-prometheus-prometheus.default:9090
  image: quay.io/prometheus/prometheus:v2.32.1
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  podMonitorNamespaceSelector: {}
  podMonitorSelector:
  matchLabels:
  release: prometheus
  portName: http-web
  probeNamespaceSelector: {}
  probeSelector:
  matchLabels:
  release: prometheus
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector:
  matchLabels:
  release: prometheus
  securityContext:
  fsGroup: 2000
  runAsGroup: 2000
  runAsNonRoot: true
  runAsUser: 1000
  serviceAccountName: prometheus-kube-prometheus-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
  matchLabels:
  release: prometheus
  shards: 1
  version: v2.32.1
```
### 3. Get the prometheus svc name 
```
PROM_SVC_NAME=$(kubectl get svc -l  app.kubernetes.io/instance=prometheus,app=kube-prometheus-stack-prometheus -ojson | jq -j '.items[].metadata.name')
```
update the hispter-shop deployment file :
```
sed -i "s,PROM_SVC_TO_REPLACE,$PROM_SVC_NAME," kubernetes-manifests/k8s-manifest.yaml
```
### 4. Deploy Tempo
```
kubectl create ns tempo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install tempo grafana/tempo -n tempo --set tempo.metricsGenerator.remoteWriteUrl="http://$PROM_SVC_NAME.default.svc.cluster.local:9090/api/v1/write" --set serviceMonitor.enabled=true
kubectl get  svc -l app.kubernetes.io/instance=tempo -n tempo -o json
TEMPO_SVC=$(kubectl get  svc -l app.kubernetes.io/instance=tempo -n tempo -o json| jq -j '.items[].metadata.name')
```
Expose the GRPC query port of Tempo
```
kubectl edit cm tempo -n tempo
```
It would be required to add `grpc_listen_port: 9095` in the `server` section
```
server:
    http_listen_port: 3100
    grpc_listen_port: 9095
```
Then we need to update the tempo service definition to expose our new port 9095
```
kubectl edit svc tempo -n tempo
```
add a new port in the Ports defintion of hte service, add:
```:
- name: tempo-grpc
  port: 9095
  protocol: TCP
  targetPort: 9095
```

### 5.Deploy OpenTelemetry Operator

#### 1. Cert-Manager
The OpenTelemetry operator requires to deploy the Cert-manager :
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```
#### 2. OpenTelemetry Operator
```
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

### 6. Configure the OpenTelemetry Collector


#### Udpate the openTelemetry manifest file
```

CLUSTERID=$(kubectl get namespace kube-system -o jsonpath='{.metadata.uid}')
CLUSTERNAME="YOUR OWN NAME"
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," otelemetry/openTelemetry-sidecar.yaml
sed -i "s,CLUSTER_NAME_TO_REPLACE,$CLUSTERNAME," otelemetry/openTelemetry-sidecar.yaml
sed -i "s,TEMP_SVC_TO_REPLACE,$TEMPO_SVC," otelemetry/openTelemetry.yaml
```
#### Deploy the OpenTelemetry Collector
```
kubectl apply -f otelemetry/rbac.yaml
kubectl apply -f otelemetry/openTelemetry.yaml
```
### 7. Deploy the hipster-shop
```
kubectl create ns hipster-shop
kubectl annotate namespaces hipster-shop 
kubectl apply -f otelemetry/openTelemetry-sidecar.yaml -n hipster-shop
kubectl apply -f kubernetes-manifests/k8s-manifest.yaml -n hipster-shop
```
### 8. Deploy Trace
```
kubectl create ns tracetest
helm repo add kubeshop https://kubeshop.github.io/helm-charts
helm repo update
helm install tracetest kubeshop/tracetest --set telemetry.dataStores.tempo.tempo.endpoint="$TEMPO_SVC.tempo.svc.cluster.local:9095" --set telemetry.exporters.collector.exporter.collector.endpoint="tracetestcollector-collector.default.svc.cluster.local:4317" --set server.telemetry.dataStore="tempo" -n tracetest --set ingress.enabled=true --set ingress.className=nginx --set 'ingress.hosts[0].host=tracetest.$IP.nip.io,ingress.hosts[0].paths[0].path=/,ingress.hosts[0].paths[0].pathType=ImplementationSpecific'
```
### 8. Open TraceTest and create a test case
Open Trace test http://tracetest.$IP.nip.io

Create a new Test case , and run a test against :
- http get http://online.$IP.nip.io/cart
<p align="center"><img src="/image/tracetest_example.png" width="40%" alt="TraceTest example" /></p>

### 9. Run A test using the CLI
#### 1.Install the CLI
```
curl -L https://raw.githubusercontent.com/kubeshop/tracetest/main/install-cli.sh | sh
```
#### 1.Configure the CLI
```
tracetest configure
```
define the url to your tracetest server
#### 2.Run a tracetest using the CLI
```
tracetest test run -d tracetest/testdefinition.yaml -w
```


