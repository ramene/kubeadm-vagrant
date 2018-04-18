## Running Ethereum and IPFS on K8s with kubeadm-vagrant

> _In [Part 1](https://gist.github.com/ramene/e918aa4664d4c40189bc2119700bf444) we laid out how we would effectively **enable** Kubernetes via docker-for-mac, edge release with kubernetes built-in.  Here, in our continued series of working sessions, we outline the manual process for those that prefer to run a X node Kubernetes Cluster locally with Kubeadm Vagrant (with VirtualBox provider)._
>
> _Don't forget to check out, [kubectl in action](https://github.com/mhausenblas/kubectl-in-action) by [Michael Hausenblas](https://github.com/mhausenblas) - `kubectl` a CLI allowing us to interact with our cluster remotely. It supports operations ranging from configuration to managing workloads and services to handle access control to administrative tasks such as node maintenance._


Part 0.1
---

> _The steps outlined are designed for Mac/Linux based CLI/Shell._

The following assumptions are made along with some links to help you get started on addressing these assumptions

1. You have a working understanding of virtualbox, vagrant and/or similar hypervisors 
2. You have a general understanding of Ethereum and Developing application on the blockchain

### Prerequisites

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [oracle virtualbox](https://www.virtualbox.org/wiki/Downloads) 
- [vagrant](https://www.vagrantup.com/downloads.html)
- vagrant-hostmanager ( `vagrant plugin install vagrant-hostmanager` )

Installation

- Clone the kubeadm-vagrant repo

```git clone https://github.com/ramene/kubeadm-vagrant ```

- Choose your distribution of choice from CentOS/Ubuntu and move to the specific directory.
- Configure the cluster parameters in Vagrantfile. Refer below for details of configuration options.

``` vi Vagrantfile ```

- Spin up the master

``` vagrant up ```

- This will spin up the Kubernetes master. You can check the status of cluster with following command,

```bash
vagrant ssh master
sudo su
kubectl get pods --all-namespaces
kubeadm token create
148a37.736fd53655b767b7
exit
exit
--> we need to set our KUBETOKEN in our Vagrantfile
```

- Spin up the nodes

set SETUP_NODES = true in Vagrantfile

``` vagrant up ```

- Copy kubeconfig to your host machine

```bash
vagrant ssh master
sudo -i
chown vagrant /etc/kubernetes/admin.conf
exit
exit
scp -P 22 -i $(vagrant ssh-config | grep -m 1 IdentityFile | cut -d ' ' -f 4) vagrant@192.168.26.10:/etc/kubernetes/admin.conf kube.config
KUBECONFIG=kube.config kubectl get nodes

NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    35m       v1.10.0
node1     Ready     <none>    30m       v1.10.0
node2     Ready     <none>    28m       v1.10.0

--> Celebrate ;-)
```

> Alternatively, if you'd like to merge your kube.config with $HOME/.kube/config do the folowing:
KUBECONFIG=$HOME/.kube/config:./kube.config kubectl config view --flatten --> config
cp $HOME/.kube/config $HOME/.kube/config_backup
cp config $HOME/.kube/config


## Good, You're still here...

As [outlined in Part 1](https://gist.github.com/ramene/e918aa4664d4c40189bc2119700bf444#first-lets-deploy-a-simple-microservice-to-our-local-kubernetes-cluster), let's deploy the [sample microservice](https://github.com/kubernauts/dok-example-us) from [Michael Hausenblas](https://github.com/mhausenblas)

  > Ok, but what else can I do?

  ``` $ KUBECONFIG=kube.config kubectl run -h | tail -n+$(kubectl run -h | grep -n Example | grep -Eo '^[^:]+') | head -n $(kubectl run -h | grep -n Options | grep -Eo '^[^:]+')```

### Post Deployment

#### _K8s Cluster Networking_

By far the more reasonable scalable and _less agonizingly painful_ solution is to use [NGINX Ingress controller](https://github.com/kubernetes/ingress-nginx/tree/nginx-0.12.0#nginx-ingress-controller) built around Kubernetes [ingress resource](http://kubernetes.io/docs/user-guide/ingress/) to surface as many Services as you wish.

_**TL;DR:**_ - [What's an Ingress Controller?](https://github.com/kubernetes/ingress-nginx/tree/nginx-0.12.0#what-is-an-ingress-controller)

```console
root@master:/home/vagrant# sudo iptables -L -n

...
Chain KUBE-EXTERNAL-SERVICES (1 references)
target     prot opt source               destination

Chain KUBE-FIREWALL (2 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes firewall for dropping marked packets */ mark match 0x8000/0x8000

Chain KUBE-FORWARD (1 references)
target     prot opt source               destination
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding rules */ mark match 0x4000/0x4000
ACCEPT     all  --  10.244.0.0/16        0.0.0.0/0            /* kubernetes forwarding conntrack pod source rule */ ctstate RELATED,ESTABLISHED
ACCEPT     all  --  0.0.0.0/0            10.244.0.0/16        /* kubernetes forwarding conntrack pod destination rule */ ctstate RELATED,ESTABLISHED
```

#### _Our current landscape might look like..._

```console
root@master:/home/vagrant# kubectl get po,no,svc --all-namespaces -o wide
NAMESPACE     NAME                                 READY     STATUS    RESTARTS   AGE       IP              NODE
kube-system   pod/etcd-master                      1/1       Running   0          18h       192.168.26.10   master
kube-system   pod/kube-apiserver-master            1/1       Running   0          18h       192.168.26.10   master
kube-system   pod/kube-controller-manager-master   1/1       Running   0          18h       192.168.26.10   master
kube-system   pod/kube-dns-86f4d74b45-p9nhv        3/3       Running   0          18h       10.244.0.2      master
kube-system   pod/kube-flannel-ds-27d5p            1/1       Running   1          18h       192.168.26.11   node1
kube-system   pod/kube-flannel-ds-pmv5s            1/1       Running   0          18h       192.168.26.10   master
kube-system   pod/kube-flannel-ds-xw5v5            1/1       Running   0          18h       192.168.26.12   node2
kube-system   pod/kube-proxy-9qcbk                 1/1       Running   0          18h       192.168.26.12   node2
kube-system   pod/kube-proxy-rwtvd                 1/1       Running   0          18h       192.168.26.11   node1
kube-system   pod/kube-proxy-vn8bp                 1/1       Running   0          18h       192.168.26.10   master
kube-system   pod/kube-scheduler-master            1/1       Running   0          18h       192.168.26.10   master
kube-system   pod/tiller-deploy-df4fdf55d-vz7lj    1/1       Running   0          18h       10.244.1.2      node1

NAME          STATUS    ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
node/master   Ready     master    18h       v1.10.1   <none>        Ubuntu 16.04.4 LTS   4.4.0-119-generic   docker://1.13.1
node/node1    Ready     <none>    18h       v1.10.1   <none>        Ubuntu 16.04.4 LTS   4.4.0-119-generic   docker://1.13.1
node/node2    Ready     <none>    18h       v1.10.1   <none>        Ubuntu 16.04.4 LTS   4.4.0-119-generic   docker://1.13.1

NAMESPACE     NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE       SELECTOR
default       service/kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP         18h       <none>
kube-system   service/kube-dns        ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP   18h       k8s-app=kube-dns
kube-system   service/tiller-deploy   ClusterIP   10.101.156.225   <none>        44134/TCP       18h       app=helm,name=tiller
```

#### _Bottom Line_

```console
$ kubectl create namespace microservices
$ kubectl -n=microservices apply -f https://raw.githubusercontent.com/kubernauts/dok-example-us/master/stock-gen/app.yaml 
$ kubectl -n=microservices apply -f https://raw.githubusercontent.com/kubernauts/dok-example-us/master/stock-con/app.yaml
```

Let's see our new pods and services
```console
root@master:/home/vagrant# kubectl get po,svc --all-namespaces -o wide
NAMESPACE       NAME                                 READY     STATUS              RESTARTS   AGE       IP              NODE
microservices   pod/stock-con-89984446c-qv2wc        0/1       ContainerCreating   0          36s       <none>          node1
microservices   pod/stock-gen-6ff64f77cd-7bbsb       0/1       ContainerCreating   0          49s       <none>          node2

NAMESPACE       NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE       SELECTOR
default         service/kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP         20h       <none>
kube-system     service/kube-dns        ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP   20h       k8s-app=kube-dns
kube-system     service/tiller-deploy   ClusterIP   10.101.156.225   <none>        44134/TCP       20h       app=helm,name=tiller
microservices   service/stock-con       ClusterIP   10.96.210.95     <none>        80/TCP          36s       app=stock-con
microservices   service/stock-gen       ClusterIP   10.110.152.206   <none>        9999/TCP        49s       app=stock-gen
```


```console
$ kubectl run webserver --image=nginx:1.13 --env="DNS_DOMAIN=cluster" --env="POD_NAMESPACE=default"
$ kubectl expose deployment webserver --port=80 --target-port=80
$ kubectl describe svc -l=run=webserver
```

_Coming in Part 2, where go over using `helm` to deploy a application stacks and build pipeline in **building DApps**._
