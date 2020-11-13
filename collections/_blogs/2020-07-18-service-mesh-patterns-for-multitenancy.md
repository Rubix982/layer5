---
layout: post
title:  "Service Mesh (Istio) patterns for Multitenancy"
date:   2020-07-18 10:30:05 -0530
image: /assets/images/posts/2020-07-18-service-mesh-patterns-for-multitenancy/image1.png
author: Sudeep Batra
tags: service mesh, open source, istio
permalink: /blog/service-mesh-patterns-for-multitenancy
---

Recently, I started learning on Service Mesh and it was a very interesting journey as I explored the Service Mesh
<img class="align-right larger" src="/assets/images/posts/2020-07-18-service-mesh-patterns-for-multitenancy/layer5-and-istio.png">
landscape starting with [Layer5 tutorials](https://github.com/layer5io/istio-service-mesh-workshop) and by exploring
the blogs at [Istio.io](https://istio.io/).

Our organization decided to use the features of Istio for securing, managing and automating microservices. However, we have to support a multi-tenant environment and that became a challenge due to lack of sufficient documentation and clarity. To give a little bit of background on the problem, we have a single cluster with multiple tenants in different namespaces sharing the same cluster . We do not want to provide each tenant their own separate Kubernetes Cluster because that would be additional provisioning overhead, management overhead and also additional resource overhead on the platform.

In this blog, I will go over the concepts and visual representation as well as the steps to learn and build your setup.
Now there are different combinations possible depending on the requirements. These could be :

1. Single Istio Control Plane with its own Ingress-Gateway to ensure traffic for the control plane is coming via the same control plane Ingress-Gateway.
1. One or more Workspace namespaces for the workload applications and each with its own instance of the ingress gateway
1. More advanced scenario would be multiple Control Plane and its associated Data Plane or essentially Multiple Service Mesh within a single Kubernetes Cluster. This is presently feasible using the Maistra Istio Operator, but its not fully supported using the upstream Istio at present. I intend to go over this use case in a later article.

Here by Ingress Gateway we mean the Istio Ingress Gateway and not the Kubernetes Ingress Gateway.
Now, before we get into the steps for building an Istio Multi-tenant setup, lets quickly review the prerequisite :

- Kubernetes setup (in the steps below I am using v1.18 on ubuntu 18.04)
  <div class="code">
  <pre>
  $ sudo apt-get install -y docker.io
  $ sudo sh -c “echo ‘deb http://apt.kubernetes.io/ kubernetes-xenial main’ >> /etc/apt/sources.list.d/kubernetes.list”
  $ sudo sh -c “curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -”
  $ sudo apt-get update
  $ sudo apt-get install -y kubeadm=1.18.1–00 kubelet=1.18.1–00 kubectl=1.18.1–00
  $ sudo kubeadm init — kubernetes-version 1.18.1 — pod-network-cidr 192.168.0.0/16
  $ mkdir -p $HOME/.kube
  $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
  $ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
  </pre> </div>

- Get the latest istioctl binary
  <div class="fit-content">
  <pre>
  $ curl -L https://git.io/getLatestIstio | sh -
  cd istio-*
  export PATH=$PWD/bin:$PATH
  istioctl version
  </pre></div>

- Initialize the Istio Operator
    <div class="code"><pre>
    $ istioctl operator init
    Using operator Deployment image: docker.io/istio/operator:1.6.3
    ✔ Istio operator installed
    ✔ Installation complete
    $ kubectl get all -n istio-operator
    NAME READY STATUS RESTARTS AGE
    pod/istio-operator-5998f6c744-vrkbk 1/1 Running 1 78m
    NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
    service/istio-operator ClusterIP 10.107.183.94 <none> 8383/TCP 78m
    NAME READY UP-TO-DATE AVAILABLE AGE
    deployment.apps/istio-operator 1/1 1 1 78m
    NAME DESIRED CURRENT READY AGE
    replicaset.apps/istio-operator-5998f6c744 1 1 1 78m

Next we will look into the details of the multitenancy scenarios; but before that a few word on the Istio operator would be nice. The Istio Operator follows the Kubernetes Controller and Custom Resource Definition mechanism and basically the above steps create an Istio operator controller in the istio-operator namespace and also registers the CRDs in the Kubernetes Registry. Basically, it extends the API for Kubernetes and allows management of a Custom Resource(CR). A word of caution, the existing upstream Istio Operator does not follow any Operator Framework like kubebuilder or Operator SDK. Hence, many of the expected behavior fail like trying to create two Istio Operator Custom Resource(the community mentions it as `iop` instances) in different namespaces or multiple `iop` instances and are under heavy development.

Ok, so now lets jump into action. We will build our Istio Setup for the scenario 1 &2 above shown in the diagram below. There are three different instances of Ingress Gateway(and it could be egress gateway as well) :

a. Ingress Gateway in Control Plane for traffic entry point to the Control plane Observability Components like `Kiali`(for the Service Mesh Graph), `Prometheus`(for metrics),Grafana(for visual charts of the metrics)and `Jaeger` (for distributed tracing between the services)
b. Ingress Gateway in the Worskpace 1 namespace for traffic entry point to the workspace 1 microservices.
c. Ingress Gateway in the Worskpace 2 namespace for traffic entry point to the workspace 2 microservices.

<div class="img-with-text">
    <img src="/assets/images/posts/2020-07-18-service-mesh-patterns-for-multitenancy/image1.png" />
    <p>Single Control Plane with Multiple Data Plane each with separate Ingress Gateway</p>
</div>

- Creation of Control Plane CR and Data Plane CR
A sample CR for creation of the Control plane and Data Plane CR can be referred below :
  <div class="fit-content">
  <pre>
  $ cat iop-platform.yaml
  apiVersion: install.istio.io/v1alpha1
  kind: IstioOperator
  metadata:
  namespace: istio-system
  name: platform-istiocontrolplane
  spec:
  profile: demo
  $ cat iop-wk1-gw.yaml
  apiVersion: install.istio.io/v1alpha1
  kind: IstioOperator
  metadata:
  namespace: istio-system
  name: wk1-gwconfig
  spec:
  profile: empty
  components:
  ingressGateways:
  — name: wk1-ingressgw
  namespace: workspace-1
  enabled: true
  label:
  istio: ingressgateway-1
  app: istio-ingressgateway-1
  values:
  gateways:
  istio-ingressgateway:
  debug: error
  $ cat iop-wk2-gw.yaml
  apiVersion: install.istio.io/v1alpha1
  kind: IstioOperator
  metadata:
  namespace: istio-system
  name: wk2-gwconfig
  spec:
  profile: empty
  components:
  ingressGateways:
  — name: wk2-ingressgw
  namespace: workspace-2
  enabled: true
  label:
  istio: ingressgateway-2
  app: istio-ingressgateway-2
  values:
  gateways:
  istio-ingressgateway:
  debug: error
  </pre></div>

It is important to make sure the indentation is correct ,otherwise it would lead to errors.
Reference: [Istio Website](https://istio.io/latest/docs/setup/install/standalone-operator/)

Deploy the CRs using kubectl:

  <div class="fit-content">
  <pre>
  $ kubectl create ns istio-system
  $ kubectl create ns workspace-1
  $ kubectl create ns workspace-2
  $ kubectl apply -f ../iop-platform.yaml
  istiooperator.install.istio.io/platform-istiocontrolplane created
  $ kubectl apply -f ../iop-wk1-gw.yaml
  istiooperator.install.istio.io/wk1-gwconfig configured
  $ kubectl apply -f ../iop-wk2-gw.yaml
  istiooperator.install.istio.io/wk2-gwconfig configured
  </pre></div>

- Troubleshooting the Istio Operator Controller.
Its a good idea to run another window and execute the below command to observe the running logs on the controller pod.
  <div class="fit-content">
  <pre>
  $ kubectl logs -f -n istio-operator
  $(kubectl get pods -n istio-operator -lname=istio-operator -o jsonpath=’{.items[0].metadata.name}’)
  </pre></div>

<img class="image-center" src="/assets/images/posts/2020-07-18-service-mesh-patterns-for-multitenancy/image2.png" />  

- Observe the created objects in istio-system and the workspace namespaces for all pods to be in running state.

  <div class="code"><pre>
  $ kubectl get all -n istio-system
  NAME READY STATUS RESTARTS AGE
  pod/grafana-b54bb57b9-k84xf 1/1 Running 0 79s
  pod/istio-egressgateway-77c7d594c5-fr79j 1/1 Running 0 83s
  pod/istio-ingressgateway-766c84dfdc-p6g6t 1/1 Running 0 83s
  pod/istio-tracing-9dd6c4f7c-ndjvn 1/1 Running 0 79s
  pod/istiod-7b69ff6f8c-9gwp8 1/1 Running 0 94s
  pod/kiali-d45468dc4–4sww2 1/1 Running 0 79s
  pod/prometheus-5fdfc44fb7–67khx 2/2 Running 0 78s
  <truncated>
  esudbat@istio:~/istio$ kubectl get all -n workspace-1
  NAME READY STATUS RESTARTS AGE
  pod/wk1-ingressgw-5674488c8b-fg2w5 1/1 Running 0 21s
  <truncated>
  esudbat@istio:~/istio$ kubectl get all -n workspace-2

- Now we will label the namespaces for istio-injection

  <div class="fit-content">
  <pre>
  $kubectl label namespace workspace-1 istio-injection=enabled
  $kubectl label namespace workspace-2 istio-injection=enabled
  </pre></div>

- Now we will deploy the bookinfo sample application in both namespaces but before that we need to make sure that the Gateway resource is updated with the correct label selector as below :

  <div class="fit-content">
  <pre>
  $ cat istio/bookinfo-gateway-1.yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
  name: bookinfo-gateway
  spec:
  selector:
  istio: ingressgateway-1 # use istio gateway-1 controller in workspace-1
  servers:
  — port:
  number: 80
  name: http
  protocol: HTTP
  hosts:
  — “*”
  $ cat istio/bookinfo-gateway-2.yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
  name: bookinfo-gateway
  spec:
  selector:
  istio: ingressgateway-2 # use istio gateway-2 controller in workspace-1
  servers:
  — port:
  number: 80
  name: http
  protocol: HTTP
  hosts:
  — “*”
  </pre></div>

Deploy the applications in workspace-1 and workspace-2 and deploy the Gateway,VirtualService and DestinationRules.

  <div class="fit-content">
  <pre>
  $kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n workspace-1
  $kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n workspace-2
  $kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n workspace-1
  $kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n workspace-2
  $kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml -n workspace-1
  $kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml -n workspace-2
  </pre></div>

Essentially what is happening is the Gateway CR is configuring the traffic to come via the ingress-gateway, it specifies the label selector based on which the ingress-gateway is selected and the host from where the traffic is allowed and port on which the traffic is allowed. It is important to ensure the label maps the label on the ingress-gateway.
Once this is done the traffic will start flowing the intended gateway and it could be observed on the Kiali dashboard.
Please note that in this scenario the Kiali is also accessed via the default control plane ingress-gateway running in the istio-system control plane namespace.

<div class="img-with-text">
    <img src="/assets/images/posts/2020-07-18-service-mesh-patterns-for-multitenancy/image3.png" />
    <p>Kiali view for Workspace-1</p>
</div>

<div class="img-with-text">
    <img src="/assets/images/posts/2020-07-18-service-mesh-patterns-for-multitenancy/image4.png" />
    <p>Kiali view for Workspace-2</p>
</div>

Shortly, we will go over the 3rd scenario which is to run multiple Istio Control Planes (Multiple Service Meshes) within the same Kubernetes Cluster. For that, we will need to build an open-shift setup and deploy the Maistra Istio Operator.
Special thanks to the [Istio Community](https://istio.slack.com/) for helping me understand the concepts and also answering my queries and of course to [Lee Calcote](https://calcotestudios.com/talks/), who helped me embark on my Istio journey.

\- _Sudeep Batra_

<div class="center"><b>Connect with Sudeep Batra on
<a href="https://www.linkedin.com/in/sudeep-batra">LinkedIn</a>, 
<a href="https://github.com/sb1975">GitHub</a>, or 
<a href="https://twitter/sudeepbatra">Twitter</a>
</b></div>