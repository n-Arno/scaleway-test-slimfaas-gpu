slimfaas@scaleway with GPU
==========================

This repo is a test of [SlimFaas](https://github.com/AxaFrance/SlimFaas) as a way to easily scale and leverage cluster autoscaling for GPU with Kapsule.

**TL;DR**: This is promising but clearly not production ready yet. The SlimFaas deployment crashed several times during tests and didn't recover by itself. Furthermore, it seems scaling up with the number of parallel calls is not implemented yet.

Also, the SlimFaas ingress configuration includes an extended timeout to handle the cluster autoscaling delay when doing a cold call to the function without waking it up.

Pre-requisites
--------------

- Kapsule cluster with a default CPU pool of one node + a GPU pool with minimum scaling at 0 (tainted using a tag: `taint=node=gpu:NoSchedule`).
- Nginx ingress controller + certmanager
- a dns entry pointing to the ingress LB IP
- (optional) `jq` CLI to parse some json in the tests

Deployment
----------

Update the dns entry in `deployment-slimfaas.yaml`

```
(snip)
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: slimfaas
  namespace: slimfaas-demo
  annotations:
    cert-manager.io/cluster-issuer: lets-encrypt
    # for the NGINX's nginx-ingress
    nginx.org/proxy-connect-timeout: 1200s
    nginx.org/proxy-read-timeout: 1200s
    nginx.org/proxy-send-timeout: 1200s
    # for the default ingress-nginx
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "1200"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "1200"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "1200"
spec:
  tls:
  - hosts:
    - <dns-entry>
    secretName: slimfaas-tls
  rules:
  - host: <dns-entry>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: slimfaas
            port:
              number: 5000
```

Install the SlimFaas deployment. This replaces a "operator/controller" and handle the scaling of the associated pods. For testing purpose we are only deploying one replica.

```
kubectl apply -f deployment-slimfaas.yaml
```

Create a sample GPU needing deployment with associated annotation

```
kubectl apply -f test-ollama.yaml
```

This will trigger the creation of a first GPU needing pod, which trigger the cluster autoscaler. Since the deployment of the node take about 10 minutes, the "function" must not scale down before that and the inactivity threshold is setup at 20 minutes.

Testing
-------

Wake up the function to trigger deployment:

```
curl -sSL https://<dns-entry>/wake-function/ollama
```

Validate a node has been added with GPU capacities

```
kubectl get nodes -ojson | jq '.items[] | select(.status.capacity | has("nvidia.com/gpu")) | .metadata.name'
```

Test the call to Ollama

```
curl -sSL https://<dns-entry>/function/ollama/api/tags | jq

curl -sSL -d '{"model":"tinyllama", "prompt": "Hello, how are you?" }' https://<dns-entry>/function/ollama/api/generate
```

Wait 20 minutes and check that the ollama pod is gone.

```
kubectl get pods -n slimfaas-demo -l app=ollama
```

Wait 15 more minutes and validate the GPU node scaled down.

```
kubectl get nodes -ojson | jq '.items[] | select(.status.capacity | has("nvidia.com/gpu")) | .metadata.name'
```

Caveat
------

SlimFaas is looking for underlaying services with a specific url format including port as defined in the deployment.

```
(snip)
          env:
            - name: BASE_FUNCTION_URL
              value: "http://{function_name}.{namespace}.svc.cluster.local:5000"
(snip)
```

Use the services to translate the application specific port to the expected port:

```
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ollama
  name: ollama
  namespace: slimfaas-demo
spec:
  ports:
  - name: ollama
    port: 5000
    protocol: TCP
    targetPort: 11434
  selector:
    app: ollama
  type: ClusterIP
```
