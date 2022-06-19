# k8s-harder-way
Building a k8s virtual cluster from scratch with Vagrant and Ansible by following https://github.com/kelseyhightower/kubernetes-the-hard-way.

## Pre reqs
Ensure the following software are installed on local machine
- vagrant
- libvirt
- ansible

## Certificates
https://kubernetes.io/docs/concepts/security/controlling-access/

https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md

Run the play `create_crypto.yml` prior to `vagrant up` to create necessary crypto objects.  This play will quickly generate a Certificate Authority and every Certificate needed to boot strap the cluster.  Certificate variables such as extended key usage, key usage, CN, etc... are all stored in `var/certs.yml` and can be tweaked if desired.  Regenerate crypto by deleting the `crypto/` directory and rerun `create_crypto.yml`.

```
ansible-playbook create_crypto.yml
```

## Provision Nodes
Bring up the nodes using `vagrant up` this will provision:

- 3 etcd nodes
- 2 controller nodes
- 2 worker nodes

The ansible play is executed after VMs are up from a high level it will:

- Add the IP/hostname of all nodes to `/etc/hosts` on all nodes.
- Copy certs for cluster components auth
- Download, configure, and start kubernetes components as systemd services(etcd, kubelet, kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy, container runtime)
- Install required packages for workers (conntrack)
- Generate kubeconfig for components to connect to the API Server
- Disable SWAP and enable port forwarding on workers

## Configure Cluster

Become root `sudo -i`

Validate the control plane is running.
```
root@controller-2:~# kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Create cluster role and cluster role binding for API Server to connect nodes.
Required for retrieving pod logs and exec
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
Install weave network and CoreDNS
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
```
kubectl apply -f https://raw.githubusercontent.com/tony-schndr/k8s-harder-way/master/files/coredns-1.8.yaml
```

Once network/coredns are up nodes should become Ready
```
root@controller-1:~# kubectl get nodes -w
NAME       STATUS     ROLES    AGE   VERSION
worker-1   NotReady   <none>   18s   v1.21.0
worker-2   NotReady   <none>   18s   v1.21.0
worker-1   NotReady   <none>   79s   v1.21.0
worker-1   Ready      <none>   80s   v1.21.0
worker-2   Ready      <none>   80s   v1.21.0
worker-1   Ready      <none>   80s   v1.21.0
worker-2   Ready      <none>   80s   v1.21.0
```

## Smoke test

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

Check kubectl port-forwarding
```
kubectl port-forward service/nginx 8080:8080
```

In another shell on the same machine `kubectl port-forward` was executed, `curl localhost:8080`

### Current issues

Pods can't resolve services external to cluster.