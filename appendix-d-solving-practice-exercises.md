# Appendix D: Solving the Practice Exercises

This appendix provides guided solutions to the practice exercises found at the end of each section, organized by chapter. Rather than jumping straight to the answers, simulate the exam environment and work through each problem as if it were a real CKA exam question.

---

## D.1 Chapter 1 Practice Exercises

### D.1.1 SWAPI API Similarities

The SWAPI (https://swapi.info/playground) is similar to the Kubernetes API:
- GET request to `https://swapi.info/api/starships/9/` returns the "Death Star" object
- Similarly, retrieving a Deployment uses: `https://kind-control-plane:6443/apis/apps/v1/namespaces/default/deployments`

### D.1.2 Listing API Resources

Use the help menu for clues:

```bash
root@kind-control-plane:/# kubectl
...
Other Commands:
  api-resources   Print the supported API resources on the server
  api-versions    Print the supported API versions on the server
...
```

**Solution:** `kubectl api-resources`

### D.1.3 Listing Background Processes

```bash
root@kind-control-plane:/# systemctl list-unit-files --type service --all | grep kube
kubelet.service                        enabled         enabled
```

### D.1.4 Status of the kubelet Service

```bash
systemctl status kubelet > kubelet-status.txt
```

### D.1.5 Using journalctl

```bash
journalctl -u kubelet
```

---

## D.2 Chapter 2 Practice Exercises

### D.2.1 Shortening the kubectl Command

```bash
# Temporary
alias k=kubectl

# Permanent
echo "alias k=kubectl" >> ~/.bashrc
```

### D.2.2 Listing Kubernetes System Pods

```bash
root@kind-control-plane:~# k get po -n kube-system -o wide
NAME                                 READY   STATUS    RESTARTS       AGE   IP            NODE
coredns-565d847f94-75lsz             1/1     Running   7 (28h ago)    56d   10.244.0.11   kind-control-plane
etcd-kind-control-plane              1/1     Running   8 (28h ago)    56d   172.18.0.2    kind-control-plane
kube-apiserver-kind-control-plane    1/1     Running   5 (28h ago)    52d   172.18.0.2    kind-control-plane
...
```

**Save to file:** `k get po -n kube-system -o wide > pod-ip-output.txt`

### D.2.3 Upgrading the Control Plane

```bash
kubeadm upgrade plan
# Shows available upgrades

kubeadm upgrade apply v1.32.6
```

> [!NOTE]
> If you get an error about kubeadm version, run: `apt update; apt install -y kubeadm=1.32.6-00`

### D.2.4 Listing PKI Directory Files

```bash
root@kind-control-plane:/# cd /etc/kubernetes/pki
root@kind-control-plane:/etc/kubernetes/pki# ls
apiserver-etcd-client.crt  apiserver.crt  ca.crt  etcd  front-proxy-ca.key  sa.pub
...
```

**Save:** `ls /etc/kubernetes/pki > certificates.txt`

### D.2.5 Viewing kubelet Client Certificate

```bash
root@kind-control-plane:/# cat /etc/kubernetes/kubelet.conf | grep client-certificate
    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem

root@kind-control-plane:/# openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout
Certificate:
    Issuer: CN = kubernetes
    Validity
        Not Before: Jun 17 18:11:48 2025 GMT
        Not After : Jun 17 18:16:48 2026 GMT
...
```

### D.2.6 Viewing CRI Implementation

```bash
root@kind-control-plane:/# crictl version
RuntimeName:  containerd
RuntimeVersion:  v2.0.2
```

### D.2.7 Listing Running Containers and Restarting apiserver

```bash
# List containers
root@kind-control-plane:/# crictl ps | egrep "CONTAINER|apiserver"
CONTAINER       IMAGE          STATE        NAME            ATTEMPT   POD ID
fa17e5196b445   73afaf82c9cc3  Running      kube-apiserver  1         664e6f4be2115

# Stop container
root@kind-control-plane:/# crictl stop fa17e5196b445

# Verify restart (new container ID, ATTEMPT increased)
root@kind-control-plane:/# crictl ps | egrep "CONTAINER|apiserver"
CONTAINER       IMAGE          STATE        NAME            ATTEMPT   POD ID
27c849fee43f0   73afaf82c9cc3  Running      kube-apiserver  2         664e6f4be2115
```

### D.2.8 Determining CNI Using kubectl

```bash
root@kind-control-plane:/# kubectl get ds -n kube-system
NAME         DESIRED   CURRENT   READY   NODE SELECTOR            AGE
kindnet      1         1         1       kubernetes.io/os=linux   7d1h
kube-proxy   1         1         1       kubernetes.io/os=linux   7d1h
```

The CNI is **Kindnet** (ignore kube-proxy).

### D.2.9 Listing StorageClasses

```bash
root@kind-control-plane:/# kubectl get storageclasses
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer
```

---

## D.3 Chapter 3 Practice Exercises

### D.3.1 Creating a Role

```bash
# Use help for examples
k create role -h

# Create role
kubectl create role sa-creator --verb=create --resource=sa
```

### D.3.2 Creating a Role Binding

```bash
kubectl create rolebinding sa-creator-binding --role=sa-creator --user=sandra
```

### D.3.3 Using auth can-i

```bash
root@kind-control-plane:~# k auth can-i create sa --as sandra
yes
```

### D.3.4 Creating a New User

```bash
# Generate private key
openssl genrsa -out sandra.key 2048

# Create CSR
openssl req -new -key sandra.key -subj "/CN=sandra/O=developers" -out sandra.csr

# Store in environment variable
export REQUEST=$(cat sandra.csr | base64 -w 0)

# Create CSR resource
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: sandra
spec:
  groups:
  - developers
  request: $REQUEST
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# Approve
kubectl certificate approve sandra

# Extract certificate
kubectl get csr sandra -o jsonpath='{.status.certificate}' | base64 -d > sandra.crt
```

### D.3.5 Adding User to kubeconfig

```bash
kubectl config set-credentials sandra --client-certificate=sandra.crt --client-key=sandra.key --embed-certs
kubectl config set-context sandra --user=sandra --cluster=kind
kubectl config use-context sandra
```

### D.3.6 Creating a Service Account with No Token Mount

```bash
kubectl create serviceaccount secure-sa
kubectl get sa secure-sa -o yaml > secure-sa.yaml
echo "automountServiceAccountToken: false" >> secure-sa.yaml
kubectl apply -f secure-sa.yaml --force

# Create pod with service account
kubectl run nginx --image nginx --dry-run=client -o yaml > pod-nomount.yaml
# Add to spec:
#   serviceAccountName: secure-sa
#   automountServiceAccountToken: false
kubectl apply -f pod-nomount.yaml
```

### D.3.7 Creating a Cluster Role

```bash
kubectl create clusterrole acme-corp-role --verb=create --resource=deploy,rs,ds
kubectl create rolebinding acme-corp-role-binding --clusterrole=acme-corp-role --serviceaccount=default:secure-sa

# Verify (should be "no" for kube-system)
kubectl -n kube-system auth can-i create deploy --as system:serviceaccount:default:secure-sa
```

---

## D.4 Chapter 4 Practice Exercises

### D.4.1 Applying a Label and Creating a Pod

```bash
# Label node
kubectl label no kind-worker disktype=ssd

# Create pod with nodeSelector
kubectl run fast --image nginx --dry-run=client -o yaml > fast.yaml
# Add to spec:
#   nodeSelector:
#     disktype: ssd
kubectl create -f fast.yaml

# Verify
kubectl get po -o wide
```

### D.4.2 Editing a Running Pod

```bash
kubectl edit po fast
# Change disktype: ssd to disktype: slow
# Save and quit - file saved to /tmp

kubectl replace -f /tmp/kubectl-edit-589741394.yaml --force
```

### D.4.3 Using Node Affinity

```yaml
# Add to pod spec:
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/os
          operator: In
          values:
          - linux
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
      preference:
        matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

### D.4.4 Creating a Pod with Limits

```yaml
spec:
  containers:
  - image: httpd
    name: pod-limited
    resources:
      limits:
        cpu: 1
        memory: 100Mi
```

### D.4.5 Creating a Deployment with Oversubscribed Requests

Request more resources than available to see pods stay Pending. Adjust values until pods fit on node.

### D.4.6 Creating a Pod with ConfigMap as Environment Variables

```bash
# Create ConfigMap
kubectl create configmap ui-data \
  --from-literal="color.good=purple" \
  --from-literal="color.bad=yellow" \
  --from-literal="allow.textmode=true"

# Pod spec:
spec:
  containers:
  - command: ["sh", "-c", "env; sleep 8200"]
    image: busybox:1.28
    name: frontend
    envFrom:
    - configMapRef:
        name: ui-data
```

---

## D.5 Chapter 5 Practice Exercises

### D.5.1 Scaling Replicas

```bash
kubectl create deploy apache --image httpd:latest
kubectl scale deploy apache --replicas=5
```

### D.5.2 Updating the Image

```bash
kubectl set image deploy apache httpd=httpd:2.4.63
kubectl get deploy apache -o yaml | grep image
```

### D.5.3 Viewing ReplicaSet Events

```bash
kubectl describe rs apache-67984dc457
```

### D.5.4 Rolling Back

```bash
kubectl rollout undo deploy apache
kubectl rollout history deploy apache
```

### D.5.5 Changing Rollout Strategy

```bash
kubectl edit deploy apache
# Change strategy.type to Recreate
```

### D.5.6 Cordoning and Uncordoning

```bash
kubectl cordon kind-worker
kubectl get no   # Shows SchedulingDisabled
kubectl uncordon kind-worker
```

### D.5.7 Removing a Taint

```bash
kubectl taint no kind-control-plane node-role.kubernetes.io/control-plane:NoSchedule-
```

### D.5.8 Installing cert-manager with Helm

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager \
  --namespace my-cert-manager \
  --create-namespace \
  --version v1.18.2 \
  --set crds.enabled=true
```

### D.5.9 Installing Argo CD

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/refs/heads/master/manifests/namespace-install.yaml
kubectl apply -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable
```

---

## D.6 Chapter 6 Practice Exercises

### D.6.1 Exec into a Pod

```bash
kubectl run nginx --image nginx
kubectl exec -it nginx -- cat /etc/resolv.conf
```

### D.6.2 Changing DNS Service

1. Edit `/etc/kubernetes/manifests/kube-apiserver.yaml` - change service-cluster-ip CIDR
2. Edit kube-dns Service to new IP
3. Delete and recreate Service

### D.6.3 Changing kubelet Configuration

Edit `/var/lib/kubelet/config.yaml`, change `clusterDNS`, then:

```bash
systemctl daemon-reload
systemctl restart kubelet
```

### D.6.4 Editing kubelet ConfigMap

```bash
kubectl -n kube-system edit cm kubelet-config
kubeadm upgrade node phase kubelet-config
systemctl daemon-reload
systemctl restart kubelet
```

### D.6.5 Scaling CoreDNS

```bash
kubectl -n kube-system scale deploy coredns --replicas 3
```

### D.6.6-D.6.10 Service and Ingress Exercises

Create Deployments, expose Services, test with curl, change to NodePort, create ingress resources.

### D.6.11 Installing Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/chadmcrowell/acing-the-cka-exam/main/ch_06/nginx-ingress-controller.yaml
```

### D.6.12 Installing CNI

See Appendix C for detailed steps on installing Calico.

### D.6.13 Setting up Network Policies

```yaml
# Default deny ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### D.6.14 Gateway API Resources

See Appendix B for Gateway API setup.

---

## D.7 Chapter 7 Practice Exercises

### D.7.1 Creating a Persistent Volume

Reference: https://mng.bz/Z9ON

```yaml
# Create PV with 100Mi, name volstore308
```

### D.7.2 Creating a PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-claim-vol
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 90Mi
```

### D.7.3 Creating a Pod with PVC

```yaml
spec:
  containers:
  - command: ["sleep", "3600"]
    image: centos:7
    volumeMounts:
    - name: vol
      mountPath: /tmp/persistence
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: pv-claim-vol
```

### D.7.4 Creating a Storage Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: node-local
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

### D.7.5-D.7.7 Additional Storage Exercises

Create PVCs for storage classes and pods with shared volumes using emptyDir.

---

## D.8 Chapter 8 Practice Exercises

### D.8.1 Fixing Pod YAML

```bash
kubectl run testbox --image busybox --command "sleep 3600"
# Pod fails - fix command format:
# command:
# - sleep
# - "3600"
kubectl replace -f /tmp/kubectl-edit-*.yaml --force
```

### D.8.2 Fixing Pod Image

```bash
kubectl run busybox2 --image busybox:1.35.0 -it -- sh
```

### D.8.3 Fixing Completed Pod

```bash
kubectl run curlpod --image=nicolaka/netshoot --command sleep --command "3600"
```

### D.8.4 Fixing Kubernetes Scheduler

```bash
# Backup
cp /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/

# Introduce error (extra 'r' in kube-schedulerr)
vim /etc/kubernetes/manifests/kube-scheduler.yaml

# Fix by removing extra 'r'
# Restore from backup if needed
```

### D.8.5 Fixing kubelet

```bash
# Check status
systemctl status kubelet

# Check config
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# Fix path from /usr/local/bin to /usr/bin

systemctl daemon-reload
systemctl restart kubelet
```

---

> [!TIP]
> Practice these exercises multiple times before the exam. The more familiar you are with these scenarios, the faster you'll complete exam tasks!
