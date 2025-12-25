# Appendix C: Installing a CNI in a kind Cluster

This appendix shows you how to install a new **Container Network Interface (CNI)** in your Kubernetes in Docker (kind) cluster. We'll install **Flannel** and **Calico**, both used for the CKA exam.

**Steps covered:**
1. Creating the kind cluster without a CNI
2. Installing the bridge CNI plugin
3. Installing either Flannel or Calico

---

## C.1 Creating a kind Cluster Without CNI

First, create a YAML file to use as input for the `kind create` command.

**Create `config-no-kindnet.yaml`:**

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
nodes:
- role: control-plane
- role: worker
```

**Create the cluster:**

```bash
$ kind create cluster --image kindest/node:v1.32.2 --config config-no-kindnet.yaml
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.32.2)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing StorageClass
 ✓ Joining worker nodes
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community
```

---

## C.2 Installing a Bridge CNI Plugin

Get a shell to both node containers and install the bridge CNI plugin.

**Step 1: Access node and install wget:**

```bash
docker exec -it kind-control-plane bash
docker exec -it kind-worker bash
# On each node:
apt update; apt install wget
```

**Step 2: Download CNI plugins:**

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.8.0/cni-plugins-linux-amd64-v1.8.0.tgz
```

**Step 3: Untar the file:**

```bash
root@kind-control-plane:/# tar -xvf cni-plugins-linux-amd64-v1.8.0.tgz
./
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth
```

**Step 4: Move bridge to the correct directory:**

```bash
mv bridge /opt/cni/bin/
```

> [!IMPORTANT]
> The `bridge` file is essential for Kubernetes to use Flannel as a CNI. It must be in `/opt/cni/bin/`.

---

## C.3 Installing Flannel CNI

After installing the bridge CNI plugin on **both** `kind-control-plane` and `kind-worker`:

**Install Flannel (from control plane):**

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**Verify nodes are ready:**

```bash
root@kind-control-plane:/# kubectl get no
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   16m   v1.32.2
kind-worker          Ready    <none>          16m   v1.32.2
```

**Verify pods:**

```bash
root@kind-control-plane:/# kubectl get po -A
NAMESPACE            NAME                                         READY   STATUS 
kube-flannel         kube-flannel-ds-d6v6t                        1/1     Running
kube-flannel         kube-flannel-ds-h7b5v                        1/1     Running
kube-system          coredns-565d847f94-txdvw                     1/1     Running
kube-system          coredns-565d847f94-vb4kg                     1/1     Running
kube-system          etcd-kind-control-plane                      1/1     Running
kube-system          kube-apiserver-kind-control-plane            1/1     Running
kube-system          kube-controller-manager-kind-control-plane   1/1     Running
kube-system          kube-proxy-9hsvk                             1/1     Running
kube-system          kube-proxy-gkvrz                             1/1     Running
kube-system          kube-scheduler-kind-control-plane            1/1     Running
local-path-storage   local-path-provisioner-684f458cdd-8bwkh      1/1     Running
```

**Flannel installation complete!**

---

## C.4 Creating a New kind Cluster

To install Calico, either:
- Delete existing cluster: `kind delete cluster`
- Create a new cluster alongside: `kind create cluster --image kindest/node:v1.32.2 --config config-no-kindnet.yaml --name cka`

**Example with new cluster named "cka":**

```bash
$ kind create cluster --image kindest/node:v1.32.2 --config config-no-kindnet.yaml --name cka
Creating cluster "cka" ...
 ✓ Ensuring node image (kindest/node:v1.32.2)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing StorageClass
 ✓ Joining worker nodes
Set kubectl context to "kind-cka"
You can now use your cluster with:

kubectl cluster-info --context kind-cka

Thanks for using kind!
```

**Access nodes with custom prefix:**

```bash
docker exec -it cka-control-plane bash
docker exec -it cka-worker bash
```

---

## C.5 Installing the Calico CNI

After installing the bridge CNI plugin (from section C.2):

**Install Calico (from control plane):**

```bash
root@cka-control-plane:/# kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/refs/tags/v3.30.3/manifests/calico.yaml
poddisruptionbudget.policy/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
serviceaccount/calico-node created
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
deployment.apps/calico-kube-controllers created
```

**Verify nodes:**

```bash
root@cka-control-plane:/# kubectl get no
NAME                STATUS   ROLES           AGE   VERSION
cka-control-plane   Ready    control-plane   61m   v1.32.2
cka-worker          Ready    <none>          61m   v1.32.2
```

**Verify pods:**

```bash
root@cka-control-plane:/# kubectl get po -A
NAMESPACE            NAME                                        READY   STATUS
kube-system          calico-kube-controllers-58dbc876ff-l9w9t    1/1     Running
kube-system          calico-node-g5h7s                           1/1     Running
kube-system          calico-node-j8g9r                           1/1     Running
kube-system          coredns-565d847f94-b6jv4                    1/1     Running
kube-system          coredns-565d847f94-mb554                    1/1     Running
kube-system          etcd-cka-control-plane                      1/1     Running
kube-system          kube-apiserver-cka-control-plane            1/1     Running
kube-system          kube-controller-manager-cka-control-plane   1/1     Running
kube-system          kube-proxy-9ss5r                            1/1     Running
kube-system          kube-proxy-dlp2x                            1/1     Running
kube-system          kube-scheduler-cka-control-plane            1/1     Running
local-path-storage   local-path-provisioner-684f458cdd-vbskp     1/1     Running
```

**Calico CNI installation complete!**
