# Appendix B: Advanced Configurations for kind

This appendix shows you how to set up your **Kubernetes in Docker (kind)** cluster for advanced configurations such as:
- Gateway API
- Custom Resource Definitions (CRDs)
- Networking configurations

**Prerequisites:** Helm and Kustomize installed with kubectl

---

## B.1 Exposing a Port in kind

This example creates a single-node kind cluster with a label and port 80 exposed.

**Create config file (macOS/Linux):**

```bash
cat <<EOF | tee config-ingress.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    listenAddress: "127.0.0.1"
  labels:
    ingress-ready: "true"

EOF
```

**Windows:**

```bash
echo "kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    listenAddress: \"127.0.0.1\"
  labels:
    ingress-ready: \"true\" > .\config-ingress.yaml
```

**Create the cluster:**

```bash
$ kind create cluster --config config-ingress.yaml
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.32.2) 
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane 
 ✓ Installing CNI
 ✓ Installing StorageClass
 ✓ Joining worker nodes 
Set kubectl context to "kind-kind"
```

**Verify:**

```bash
$ kubectl get no --show-labels && docker port kind-control-plane
NAME                STATUS   ROLES           AGE    VERSION   LABELS
kind-control-plane  Ready    control-plane   108s   v1.32.2   
↪beta.kubernetes.io/arch=arm64,...,ingress-ready=true,...
80/tcp -> 0.0.0.0:80
6443/tcp -> 127.0.0.1:59897
```

Now you have a single-node cluster that can accept traffic over port 80 with the label `ingress-ready=true`.

> [!NOTE]
> If you already have a kind cluster running, give the new cluster a different name using `--name`. Example: `kind create cluster --config config-ingress.yaml --name ingress`

---

## B.2 Installing Gateway API

### Create the Cluster

Create a new kind cluster with port mapping for Gateway API:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30110
        hostPort: 30110
        protocol: TCP
```

```bash
$ kind create cluster --config config-gwapi.yaml
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.32.2) 
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane 
 ✓ Installing CNI
 ✓ Installing StorageClass
 ✓ Joining worker nodes 
Set kubectl context to "kind-kind"
```

> [!NOTE]
> To install the Gateway API CRDs, you need git installed: `apt update && apt install -y git`

---

### Install Gateway API CRDs

Gateway API CRDs define the API surface for managing ingress traffic. Install using Kustomize:

```bash
kubectl apply -k "github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref=v1.2.1"
```

**Output:**

```
customresourcedefinition.apiextensions.k8s.io/backendlbpolicies.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/backendtlspolicies.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/tcproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/tlsroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/udproutes.gateway.networking.k8s.io created
```

---

### Install Contour Gateway Controller

Contour is a gateway controller that implements the logic for those CRDs. Any compatible controller works (NGINX Gateway Fabric, Istio, GKE Gateway, etc.).

**Add Helm repository and install:**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install contour bitnami/contour --set gateway.enabled=true --set envoy.service.type=NodePort
```

**Output:**

```
NAME: contour
LAST DEPLOYED: Fri Jun 20 11:09:23 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: contour
CHART VERSION: 21.0.5
APP VERSION: 1.32.0
```

**Verify pods:**

```bash
root@kind-control-plane:/# kubectl -n default get po
NAME                               READY   STATUS    RESTARTS   AGE
contour-contour-7ff9f6fffc-5jxkc   1/1     Running   0          110s
contour-envoy-7gscd                2/2     Running   0          110s
```

| Pod | Purpose |
|-----|---------|
| `contour-contour-*` | Watches Gateway API resources, translates to envoy config |
| `contour-envoy-*` | Listens on HTTP/HTTPS ports, handles incoming requests |

---

### Create GatewayClass and Gateway

**Create GatewayClass:**

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: contour
spec:
  controllerName: projectcontour.io/gateway-controller
EOF
```

**Create Gateway:**

```bash
kubectl apply -f - <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: contour
  namespace: default
spec:
  gatewayClassName: contour
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
EOF
```

**Verify:**

```bash
root@kind-control-plane:/# kubectl get gateways
NAMESPACE        NAME      CLASS     ADDRESS   PROGRAMMED   AGE
default          contour   contour             Unknown      81s
```

---

### Create Deployment, Service, and HTTPRoute

**Create Deployment:**

```bash
kubectl create deploy echo-server -n default --image=ealen/echo-server --port 80
```

**Create Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: echo-server
  name: echo-server
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30110
  type: NodePort
  selector:
    app: echo-server
```

**Create HTTPRoute:**

```bash
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: echo-route
  namespace: default
spec:
  parentRefs:
  - name: contour
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: echo-server
      port: 80
EOF
```

---

### Test the HTTPRoute

```bash
root@kind-control-plane:/# curl http://localhost:30110
{"host":{"hostname":"localhost","ip":"::ffff:10.244.0.1","ips":[]},
"http":{"method":"GET","baseUrl":"","originalUrl":"/","protocol":
"http"},"request":{"params":{"0":"/"},"query":{},"cookies":{},
"body":{},"headers":{"host":"localhost:30110","user-agent":
"curl/7.88.1","accept":"*/*"}},"environment":…
```

> [!TIP]
> For human-readable output, use: `curl http://localhost:30110 | jq`
