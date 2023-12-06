# Deploy Kuma-Demo

1. Escalate scc of postgres SA to `privileged` (this is for demo purposes only, not recommended for production):

  ```console
  oc adm policy add-scc-to-user privileged -z kuma-demo-postgres -n kuma-demo
  ```

2. Escalate SCC of the default SA (this is the SA for the rest of the kuma-demo application pods) to use the kong-mesh-sidecar SCC:
  
  ```console
  oc adm policy add-scc-to-user kong-mesh-sidecar -z default -n kuma-demo
  ```

3. Deploy the application (if not in the kuma-demo directory, change to it)

  ```console
  kubectl apply -f kuma-demo-aio-ocp.yaml
  ```

4. You should see the following output (4 application pods, each with 2 containers):

  ```console
  kuma-demo  $ oc get pods
  NAME                                    READY   STATUS    RESTARTS   AGE
  kuma-demo-app-5f5f455864-7bn5k          1/2     Running   0          9s
  kuma-demo-backend-v0-68bc879754-hck4t   1/2     Running   0          10s
  postgres-master-6576d699c8-f5wjn        2/2     Running   0          9s
  redis-master-946d4d567-8fvn5            2/2     Running   0          10s
  ```

5. Use port-forward to view the application:

  ```console
  kubectl port-forward service/frontend -n kuma-demo 8080
  ```