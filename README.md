# Kong Mesh Quickstart for Openshift 4.12


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

oc login https://api.df-mesh-2.slzk.p1.openshiftapps.com:6443 --username cluster-admin --password RYvrE-6tUWv-CrWnD-ZzEyq

```console
kubectl create namespace kong-mesh-system
```

Create the image pull secret:

```console
oc create secret docker-registry rh-registry-secret -n kong-mesh-system \
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

Thanks for making it to the end!