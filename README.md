# Kong Mesh Quickstart for Openshift 4.12

<p align="center">
  <img src="https://konghq.com/wp-content/uploads/2018/08/kong-combination-mark-color-256px.png" /></div>
</p>

This is quickstart to get you up and running with Kong Mesh in standalone mode on Openshift.

This tutorial does not require a license, Kong Mesh can start in evaluation mode, and limits you 5 dataplanes/sidecars to use. Just enough dataplanes to get comfortable with the product.

For Openshfit, we'll be spinning up a ROSA 4.12 cluster. This tutorial does also assume some base level knowledge of Openshift.

This tutorial will cover:

* How to use the Red Hat Certified Kong Mesh Images
* How to implement the required openshift scc for the kong-mesh sidecar
* Deploy and join the Kong for Kubernetes Ingress Controller (KIC) to the mesh
* Deploy a sample application, bookinfo, on the mesh and validate it's all working

Fun! Let's do it!

## Table of Contents

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=true} -->

<!-- code_chunk_output -->

* [Prequisites](#prequisites)
* [Install ROSA](#install-rosa)
* [Install Kong Mesh](#install-kong-mesh---standalone-mode)
* [Deploy KIC](#deploy-kic-on-mesh)
* [Deploy Bookinfo](#deploy-bookinfo)

<!-- /code_chunk_output -->

## Prequisites

The prequisites for this tutorial:

1. ROSA cli or another Openshift 4.12 cluster with the ability to create LoadBalancer type Kubernetes Services
2. kubectl cli
3. oc cli
4. Helm 3

## Install ROSA

Create System Variables:

```console
CLUSTER_NAME=df-mesh-2
REGION=us-west-2
```

Create a small ROSA cluster:

```console
rosa create cluster --cluster-name=$CLUSTER_NAME --region=$REGION --multi-az=false --version 4.12.3
```

When the Cluster install is complete, create a cluster-admin user:

```console
rosa create admin --cluster $CLUSTER_NAME
```

Validate you can login to the cluster via the credentials provided by the rosa cli stdout. Once login is successful you can proceed to the next step.

## Install Kong Mesh - Standalone Mode

```console
kubectl create namespace kong-mesh-system
```

Create the image pull secret:

```console
kubectl create secret docker-registry rh-registry-secret -n kong-mesh-system \
    --docker-server=registry.connect.redhat.com \
    --docker-username=<username> \
    --docker-password=<password> \
    --docker-email=<email>
```

Add nonroot-v2 to job service accounts:

```console
oc adm policy add-scc-to-user nonroot-v2 system:serviceaccount:kong-mesh-system:kong-mesh-install-crds
oc adm policy add-scc-to-user nonroot-v2 system:serviceaccount:kong-mesh-system:kong-mesh-patch-ns-job 
oc adm policy add-scc-to-user nonroot-v2 system:serviceaccount:kong-mesh-system:kong-mesh-pre-delete-job
```

Grab the latest helm chart:

```console
helm repo add kong-mesh https://kong.github.io/kong-mesh-charts
```

Install Kong Mesh

```console
helm upgrade -i kong-mesh kong-mesh/kong-mesh --version 2.2.0 -f kong-mesh/values.yaml -n kong-mesh-system
```

Open a second terminal and port-forward to reach the mesh GUI:

```console
kubectl port-forward svc/kong-mesh-control-plane -n kong-mesh-system 5681:5681
```

You should be able to reach the Kong-Mesh UI at `http://localhost:5681/gui`

Last, do some prep work for the sidecar itself so sidecars will startup successfully. Apply the kong-mesh-sidecar scc and corresponding container patches:

```console
kubectl create -f kong-mesh/kong-mesh-sidecar-scc.yaml
kubectl apply -f kong-mesh/container-patch.yaml 
```

## Deploy KIC on Mesh

First we're going to deploy KIC as our ingress controller:

```console
kubectl create namespace kong 
kubectl label namespace kong kuma.io/sidecar-injection=enabled

oc adm policy add-scc-to-user kong-mesh-sidecar system:serviceaccount:kong:kong-kong

helm install kong kong/kong -n kong \
--set ingressController.installCRDs=false \
--set podAnnotations."kuma\.io/mesh"=default \
--set podAnnotations."kuma\.io/gateway"=enabled
```

export PROXY_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].hostname}" service -n kong kong-kong-proxy)

curl -i $PROXY_IP

The expected output is:

```console
HTTP/1.1 404 Not Found
Date: Tue, 02 May 2023 15:40:05 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 48
X-Kong-Response-Latency: 0
Server: kong/3.2.2

{"message":"no Route matched with those values"}%
```

## Deploy Bookinfo

Allow all the service accounts to use the kong-mesh-sidecar scc:

```console
oc adm policy add-scc-to-user kong-mesh-sidecar system:serviceaccount:bookinfo:bookinfo-details
oc adm policy add-scc-to-user kong-mesh-sidecar system:serviceaccount:bookinfo:bookinfo-productpage
oc adm policy add-scc-to-user kong-mesh-sidecar system:serviceaccount:bookinfo:bookinfo-ratings
oc adm policy add-scc-to-user kong-mesh-sidecar system:serviceaccount:bookinfo:bookinfo-reviews
```

Next, deeploy bookinfo. This bookinfo app has been paired down to work with Kong Mesh in evaluation mode. We only have 5 sidecars we can deploy on evaluation mode is all.

```console
kubectl apply -f bookinfo/bookinfo.yaml
```

Last we want to create an ingress resource to expose bookinfo to the outside world.

```console
kubectl apply -f bookinfo/ingress-productpage.yaml
```

Now! Validate it's all up and running. Navigate to your browswer to `http://$PROXY_IP/productpage` and you should be able to see productpage with a majority of the bells and whistles.

## Clean Up

Tear Down bookinfo:

```console
kubectl delete deploy,svc,ingress --all -n bookinfo
```

Tear Down KIC:

```console
helm uninstall kong -n kong
```

Last Tear Down Kong Mesh

```console
helm uninstall kong-mesh -n kong-mesh-system
```

And all the components should be down! It's safe to destroy the ROSA cluster.

Delete the ROSA cluster-admin user:

```console
rosa delete admin --cluster $CLUSTER_NAME
```

Delete ROSA cluster:

```console
rosa delete cluster --cluster $CLUSTER_NAME
```

Thanks for making it to end!