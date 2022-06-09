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

## Worker bootsrap

Validate the worker nodes are running.
```
root@controller-1:~# kubectl get nodes --kubeconfig /etc/kubernetes/config/admin.kubeconfig
NAME       STATUS   ROLES    AGE   VERSION
worker-1   Ready    <none>   13m   v1.21.0
worker-2   Ready    <none>   13m   v1.21.0
```
## Set admin.kubeconfg
```
export KUBECONFIG=/etc/kubernetes/config/admin.kubeconfig
```
## Role Bindings

Create role binding for API Server to kubelet
```
cat <<EOF | kubectl apply -f -
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

cat <<EOF | kubectl apply -f -
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
## Network addon

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
## Install core DNS Addon

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.8.yaml --kubeconfig /etc/kubernetes/config/admin.kubeconfig
```


Smoke test
---

Create a deployment and scale it to 2 + pods, check that pods are deployed between both workers
```
root@controller-1:~# kubectl create deployment nginx --image=nginx:latest
deployment.apps/nginx created

root@controller-1:~# kubectl scale deployment nginx --replicas=5
deployment.apps/nginx scaled

root@controller-1:~# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
nginx-7c658794b9-5fl2x   1/1     Running   0          44s   10.32.0.5   worker-1   <none>           <none>
nginx-7c658794b9-87s8p   1/1     Running   0          89s   10.44.0.1   worker-2   <none>           <none>
nginx-7c658794b9-9qg8g   1/1     Running   0          44s   10.32.0.4   worker-1   <none>           <none>
nginx-7c658794b9-hn7dq   1/1     Running   0          44s   10.44.0.3   worker-2   <none>           <none>
nginx-7c658794b9-nn54d   1/1     Running   0          44s   10.44.0.2   worker-2   <none>           <none>
```

Expose the nginx deployment
```
root@controller-1:~# kubectl expose deployment  nginx --target-port 80 --port 8080
service/nginx exposed
```

Check network connectivity, run this pod that has network troubleshooting tools installed. 
```
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
```

Once at the bash prompt in the helper pod, curl the nginx service, validate html is returned.
```
bash-5.1# curl nginx:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Current issues
