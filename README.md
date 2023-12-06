# Kong Mesh Quickstart for OpenShift 4.12

<p align="center">
  <img src="https://konghq.com/wp-content/uploads/2018/08/kong-combination-mark-color-256px.png" /></div>
</p>

This is a quickstart to get you up and running with Kong Mesh in standalone mode on OpenShift.

This tutorial does not require a license, Kong Mesh can start in evaluation mode, and limits you 5 dataplanes/sidecars to use. Just enough dataplanes to get comfortable with the product.

For OpenShfit, we'll be spinning up a ROSA 4.12 cluster. This tutorial does also assume some base level knowledge of OpenShift.

This tutorial will cover:

* How to use the Red Hat Certified Kong Mesh Images
* How to implement the required OpenShift SecurityContextConstraints (SCC) for the kong-mesh sidecar
* Deploy and join the Kong for Kubernetes Ingress Controller (KIC) to the mesh
* Deploy a sample application, bookinfo, on the mesh and validate it's all working

Fun! Let's do it!

## Table of Contents

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=true} -->

<!-- code_chunk_output -->

* [Prerequisites](#prerequisites)
* [Install ROSA](#install-rosa)
* [Install Kong Mesh](#install-kong-mesh---standalone-mode)
* [Deploy Bookinfo](bookinfo/README.md)
* [Deploy Kuma-Demo](kuma-demo/README.md)
* [Clean Up](#clean-up)

<!-- /code_chunk_output -->

## Prerequisites

The prerequisites for this tutorial:

1. ROSA cli or another OpenShift 4.12 cluster with the ability to create LoadBalancer type Kubernetes Services
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
rosa create cluster --cluster-name=$CLUSTER_NAME --region=$REGION --multi-az=false --version 4.12.13
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

From here, you can proceed to the [Bookinfo](bookinfo/README.md) tutorial or the [Kuma-Demo](kuma-demo/README.md) tutorial to test out Kong Mesh.

## Clean Up

Tear Down bookinfo:

```console
kubectl delete deploy,svc,ingress --all -n bookinfo
```

Tear Down KIC:

```console
helm uninstall kong -n kong
```

Tear Down Kuma-Demo:

```console
kubectl delete deploy,svc --all -n kuma-demo
```

Last Tear Down Kong Mesh:

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