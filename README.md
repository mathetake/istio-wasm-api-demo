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
$ kubectl run curl --restart=OnFailure -l sidecar.istio.io/inject=false --image=curlimages/curl -it --rm -- /bin/sh -c 'curl -v http://httpbin.default.svc.cluster.local:8000/headers'

*   Trying 10.96.158.134:8000...
* Connected to httpbin.default.svc.cluster.local (10.96.158.134) port 8000 (#0)
> GET /headers HTTP/1.1
> Host: httpbin.default.svc.cluster.local:8000
> User-Agent: curl/7.80.0-DEV
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< server: istio-envoy
< date: Mon, 15 Nov 2021 05:25:51 GMT
< content-type: application/json
< content-length: 259
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 1
< x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*
<
{
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.default.svc.cluster.local:8000",
    "User-Agent": "curl/7.80.0-DEV",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "673a180f34d0082f",
    "X-B3-Traceid": "8d58033071c3d249673a180f34d0082f"
  }
}
* Connection #0 to host httpbin.default.svc.cluster.local left intact
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
$ docker build -t ${WASM_EXTENSION_REGISTRY} .
```

3. Publish the docker image to your OCI registry

```
# Make sure you already logged in to the registory.
docker push ${WASM_EXTENSION_REGISTRY}
```
