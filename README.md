# istio-wasm-api-demo

## Setup the latest Istio

0. Setup k8s cluster: e.g. `kind create cluster --name test`

1. Download the latest Istioctl from the GitHub releases (as the Wasm API is not officially released yet):

E.g. https://github.com/istio/istio/releases/tag/1.12.0-beta.2

2. Install Istio demo profile:

```
/path/to/istio-1.12.0-beta.2/bin/istioctl install --set profile=demo
```

3. Enable sidecar injection in default namespace

```
kubectl label namespace default istio-injection=enabled
```

4. Deploy the demo app/
```
echo 'apiVersion: v1
kind: ServiceAccount
metadata:
  name: httpbin
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
    service: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      serviceAccountName: httpbin
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 80' | kubectl apply -f -
```
