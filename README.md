# k8-harder-way

Building a k8 virtual cluster from scratch with Vagrant and Ansible by following https://github.com/kelseyhightower/kubernetes-the-hard-way.

## Setup

Ensure vagrant, virtual box, and ansible are installed on the host.  This project was built on `Ubuntu 20.04.3 LTS`

Install ansible in a virtual environment from devel branch.
```
python3 -m venv venv
. ./venv/bin/activate
pip install https://github.com/ansible/ansible/archive/devel.tar.gz --disable-pip-version-check
```

---
## Certificates
https://kubernetes.io/docs/concepts/security/controlling-access/

https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md

Run the play `create_crypto.yml` prior to `vagrant up` to create necessary crypto objects.  This play will quickly generate a Certificate Authority and every Certificate needed to boot strap the cluster.  Certificate variables such as extended key usage, key usage, CN, etc... are all stored in `var/certs.yml` and can be tweaked if desired.  When regenerate crypto delete the `crypto/` directory and rerun `create_crypto.yml`, rinse and repeat if needed.  To reapply crypto settings to the cluster run `vagrant provision`.

```
ansible-playbook create_crypto.yml
```
---
## Kubernetes configuration files for auth
https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md

## etcd bootstrap
Create certificates with SANs for all possible ways cluster members to call each other ie DNS and IP. For simplicity one cert is used between all etcd cluster members.

SANs Example:
```
X509v3 Subject Alternative Name:
                DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster, DNS:kubernetes.svc.cluster.local, IP Address:10.32.0.1, IP Address:10.240.0.21, IP Address:10.240.0.22, IP Address:10.240.0.11, IP Address:10.240.0.12, IP Address:10.240.0.13, DNS:localhost, IP Address:127.0.0.1, DNS:etcd-1, DNS:etcd-2, DNS:etcd-3
```
Extended Key Usage `serverAuth` and `clientAuth`

Distribute the cert/key pair and CA chain to all etcd nodes.  Create `etcd.service` on all nodes, this is templated in `templates/etcd.service.j2`.


## Controller bootstrap

Validate the control plane is running.
```
root@controller-2:~# kubectl cluster-info --kubeconfig /etc/kubernetes/config/admin.kubeconfig
Kubernetes control plane is running at https://127.0.0.1:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```bash
cat <<EOF | kubectl apply --kubeconfig /etc/kubernetes/config/admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF

```

```bash
cat <<EOF | kubectl apply --kubeconfig /etc/kubernetes/config/admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## Worker bootsrap

Validate the worker nodes are running.
```
root@controller-1:~# kubectl get nodes --kubeconfig /etc/kubernetes/config/admin.kubeconfig
NAME       STATUS   ROLES    AGE   VERSION
worker-1   Ready    <none>   13m   v1.21.0
worker-2   Ready    <none>   13m   v1.21.0
```
```
system:kube-proxy

cat <<EOF | kubectl apply --kubeconfig /etc/kubernetes/config/admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: kube-proxy-role
rules:
  -
    apiGroups:
      - ""
    resources:
      - endpoints
      - events
      - services
      - nodes
    verbs: ["get", "watch", "list"]
  - nonResourceURLs: ["*"]
    verbs: ["get", "watch", "list"]
  -
    apiGroups:
      - ""
    resources:
      - events
    verbs: ["*"]
  - nonResourceURLs: ["*"]
    verbs: ["*"]
EOF

```
```
cat <<EOF | kubectl apply --kubeconfig /etc/kubernetes/config/admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-proxy
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-proxy-role
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

kubectl config set-cluster kubernetes-the-hard-way \
--certificate-authority=crypto/ca.pem \
--embed-certs=true \
--server=https://10.240.0.21:6443

kubectl config set-credentials admin \
--client-certificate=crypto/admin.pem \
--client-key=crypto/admin-key.pem

kubectl config set-context kubernetes-the-hard-way \
--cluster=kubernetes-the-hard-way \
--user=admin

kubectl config use-context kubernetes-the-hard-way

```


## Install core DNS Addon

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.8.yaml --kubeconfig /etc/kubernetes/config/admin.kubeconfig
```
