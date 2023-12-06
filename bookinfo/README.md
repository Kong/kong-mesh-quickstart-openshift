## Deploy KIC on Mesh

First we're going to deploy KIC as our ingress controller:

```console
kubectl create namespace kong 
kubectl label namespace kong kuma.io/sidecar-injection=enabled
oc adm policy add-scc-to-user kong-mesh-sidecar system:serviceaccount:kong:kong-kong
```

Grab the latest chart:

```console
helm repo add kong https://charts.konghq.com
helm repo update
```

Then run the install:

```console
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

Next, deploy bookinfo. This bookinfo app has been paired down to work with Kong Mesh in evaluation mode. We only have 5 sidecars we can deploy on evaluation mode is all.

```console
kubectl create namespace bookinfo

kubectl label namespace bookinfo kuma.io/sidecar-injection=enabled
```

```console
kubectl apply -f bookinfo/bookinfo.yaml -n bookinfo
```

Last we want to create an ingress resource to expose bookinfo to the outside world.

```console
kubectl apply -f bookinfo/ingress-productpage.yaml -n bookinfo
```

Now! Validate it's all up and running. Navigate to your browswer to `http://$PROXY_IP/productpage` and you should be able to see productpage with a majority of the bells and whistles.
