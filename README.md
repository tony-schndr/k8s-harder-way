# k8-harder-way

Building a k8 bare metal cluster from scratch with Vagrant and Ansible by following https://github.com/kelseyhightower/kubernetes-the-hard-way

---
## Certificates
https://kubernetes.io/docs/concepts/security/controlling-access/

https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md

Run play to create certs from the variables defined in `var/certs.yml`
```
ansible-playbook create_certs.yml
```

When vagrant machines are provisioned these certs are uploaded to the appropriate VMs.

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


## control plane bootstrap
