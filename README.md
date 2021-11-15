# istio-wasm-api-demo

## 1. Setup the latest Istio

0. Setup k8s cluster: e.g. `kind create cluster --name test`

1. Download the latest Istioctl from the GitHub releases (as the Wasm API is not officially released yet):

E.g. https://github.com/istio/istio/releases/tag/1.12.0-beta.2

2. Install Istio demo profile:

```
$ /path/to/istio-1.12.0-beta.2/bin/istioctl install --set profile=demo
```

3. Enable sidecar injection in default namespace

```
$ kubectl label namespace default istio-injection=enabled
```

4. Deploy the demo app/
```
$ echo 'apiVersion: v1
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

5. Verify the demo app is running!

```
$ kubectl run curl --restart=OnFailure -l sidecar.istio.io/inject=false --image=curlimages/curl -it --rm -- /bin/sh -c 'curl --head http://httpbin.default.svc.cluster.local:8000/headers'

HTTP/1.1 200 OK
server: istio-envoy
date: Mon, 15 Nov 2021 05:54:45 GMT
content-type: application/json
content-length: 259
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 1
x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*

pod "curl" deleted
```

## 2. Build and publish Wasm extension on OCI registry


Dependency:: TinyGO, and Docker CLI


1. Compile the code to Wasm binary.

```
$ tinygo build -o main.wasm -scheduler=none -target=wasi main.go
```

2. Build docker image which is compliant with Wasm OCI image spec (https://github.com/solo-io/wasm/tree/master/spec)

```
# Note that replace ${WASM_EXTENSION_REGISTRY} with your OCI repo.
# Here I push to GitHub Container Registry.
$ export WASM_EXTENSION_REGISTRY=ghcr.io/mathetake/wasm-extension-demo
$ docker build -t ${WASM_EXTENSION_REGISTRY}:v1 .
```

3. Publish the docker image to your OCI registry

```
# Make sure you already logged in to the registory.
docker push ${WASM_EXTENSION_REGISTRY}:v1
```


## 3. Deploy the Wasm extension via Istio Wasm Plugin API

1. Apply the new Wasm Plugin API pointing to the Docker image.

```
$ echo 'apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: header-injection
  namespace: default
spec:
  selector:
    matchLabels:
      app: httpbin
  url: oci://ghcr.io/mathetake/wasm-extension-demo:v1
' | kubectl apply -f -
```

2. Check if the header injection works!

```dotnetcli
$ kubectl run curl --restart=OnFailure -l sidecar.istio.io/inject=false --image=curlimages/curl -it --rm -- /bin/sh -c 'curl --head http://httpbin.default.svc.cluster.local:8000/headers'

HTTP/1.1 200 OK
server: istio-envoy
date: Mon, 15 Nov 2021 05:55:19 GMT
content-type: application/json
content-length: 259
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 1
who-am-i: wasm-extension # <------------------- injected by Wasm extension!!
injected-by: istio-api!  # <------------------- injected by Wasm extension!!
x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*

pod "curl" deleted
```
