## Running Ethereum and IPFS on K8s with kubeadm-vagrant and helm

> _In [Part 1](https://gist.github.com/ramene/e918aa4664d4c40189bc2119700bf444) we laid out how we would effectively **enable** Kubernetes via docker-for-mac, edge release with kubernetes built-in.  Here, I outline the manual process for those that prefer to run a X node kubernetes cluster locally with kubeadm+vagrant, also including Helm._
>
> _Don't forget to check out, [kubectl in action](https://github.com/mhausenblas/kubectl-in-action) by [Michael Hausenblas](https://github.com/mhausenblas) - `kubectl` a CLI allowing us to interact with our cluster remotely. It supports operations ranging from configuration to managing workloads and services to handle access control to administrative tasks such as node maintenance._


Part 0.1
---

> _The steps outlined are designed for Mac/Linux based CLI/Shell._

The following assumptions are made

1. You have a working understanding of virtualbox, vagrant and/or similar hypervisors 
2. You have a general understanding of Ethereum and Developing application on the blockchain

### Prerequisites

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [oracle virtualbox](https://www.virtualbox.org/wiki/Downloads) 
- [vagrant](https://www.vagrantup.com/downloads.html)
- vagrant-hostmanager ( `vagrant plugin install vagrant-hostmanager` )

### What all we're deploying

Kubernetes works in server-client setup, where it has a master providing centralized control for a number of nodes. We will be deploying a Kubernetes master with 2 nodes.

Kubernetes has several components, outlined below:

    1. etcd - A highly available key-value store for shared configuration and service discovery.
    2. flannel - An etcd backed network fabric for containers.
    3. kube-apiserver - Provides the API for Kubernetes orchestration.
    4. kube-controller-manager - Enforces Kubernetes services.
    5. kube-scheduler - Schedules containers on hosts.
    6. kubelet - Processes a container manifest so the containers are launched according to how they are described.
    7. kube-proxy - Provides network proxy services.

Installation

- Clone the kubeadm-vagrant repo

```git clone https://github.com/ramene/kubeadm-vagrant ```

- Choose your distribution of choice from CentOS/Ubuntu and move to the specific directory.
- Configure the cluster parameters in Vagrantfile. Refer below for details of configuration options.

``` vi Vagrantfile ```

- Spin up the master

``` vagrant up ```

> This will spin up the Kubernetes master. You can check the status of cluster with following command,

```bash
vagrant ssh master
sudo su
kubectl get po,svc --all-namespaces -o wide
helm init
kubeadm token create
148a37.736fd53655b767b7
```

> Let's modify our Vagrantfile once again, taking note of the token created in the previous step

```console
SETUP_NODES = true
KUBETOKEN = "<token>"
```

``` vagrant up ```

> Copy kubeconfig to your host machine

```console
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
```

> _Recommended, if you're working across multiple clusters, merge your kube.config with $HOME/.kube/config by running the following_

```console
$ KUBECONFIG=$HOME/.kube/config:./kube.config kubectl config view --flatten --> config
$ cp $HOME/.kube/config $HOME/.kube/config_backup
$ cp config $HOME/.kube/config
```


### Part 0.2 - Post Deployment

As [outlined in Part 1](https://gist.github.com/ramene/e918aa4664d4c40189bc2119700bf444#part-1---leveraging-kubernetes-and-cloud-native-microservices), let's deploy the same example [cloud native microservice](https://github.com/kubernauts/dok-example-us) from [Michael Hausenblas](https://github.com/mhausenblas)

> _Want to see more?, run the following either on the `master` node or on the host_ .

  `$ KUBECONFIG=kube.config kubectl run -h | tail -n+$(kubectl run -h | grep -n Example | grep -Eo '^[^:]+') | head -n $(kubectl run -h | grep -n Options | grep -Eo '^[^:]+')`
  
> Open a shell to a running container
    see: https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/

#### _Kubernetes Cluster Networking_

By far the more reasonable scalable and _less agonizingly painful_ solution is to use [NGINX Ingress controller](https://github.com/kubernetes/ingress-nginx/tree/nginx-0.12.0#nginx-ingress-controller) built around Kubernetes [ingress resource](http://kubernetes.io/docs/user-guide/ingress/) to surface as many services as you wish.

_**TL;DR:**_ - [What's an Ingress Controller?](https://github.com/kubernetes/ingress-nginx/tree/nginx-0.12.0#what-is-an-ingress-controller)

#### Talking to our POD network

```
$ sysctl net.bridge.bridge-nf-call-iptables=1
$ export kubever=$(kubectl version | base64 | tr -d '\n')
$ kubectl apply -f "https://gist.github.com/ramene/13db280174bfb9441f813e83a0d149da"
```

#### _Get the lay of the land, specific to kubernetes..._

```console
root@master:/home/vagrant# sudo iptables -L -n
...
root@master:/home/vagrant# sudo iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-EXTERNAL-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes externally-visible service portals */
KUBE-FIREWALL  all  --  0.0.0.0/0            0.0.0.0/0

Chain FORWARD (policy DROP)
target     prot opt source               destination
WEAVE-NPC  all  --  0.0.0.0/0            0.0.0.0/0            /* NOTE: this must go before '-j KUBE-FORWARD' */
NFLOG      all  --  0.0.0.0/0            0.0.0.0/0            state NEW nflog-group 86
DROP       all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
KUBE-FORWARD  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding rules */
DOCKER-ISOLATION  all  --  0.0.0.0/0            0.0.0.0/0
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  172.17.0.0/16        0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            172.17.0.0/16

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes service portals */
KUBE-FIREWALL  all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER (1 references)
target     prot opt source               destination

Chain DOCKER-ISOLATION (1 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain KUBE-EXTERNAL-SERVICES (1 references)
target     prot opt source               destination

Chain KUBE-FIREWALL (2 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes firewall for dropping marked packets */ mark match 0x8000/0x8000

Chain KUBE-FORWARD (1 references)
target     prot opt source               destination
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding rules */ mark match 0x4000/0x4000
ACCEPT     all  --  172.17.0.0/16        0.0.0.0/0            /* kubernetes forwarding conntrack pod source rule */ ctstate RELATED,ESTABLISHED
ACCEPT     all  --  0.0.0.0/0            172.17.0.0/16        /* kubernetes forwarding conntrack pod destination rule */ ctstate RELATED,ESTABLISHED

Chain KUBE-SERVICES (1 references)
target     prot opt source               destination

Chain WEAVE-NPC (1 references)
target     prot opt source               destination
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
ACCEPT     all  --  0.0.0.0/0            224.0.0.0/4
WEAVE-NPC-DEFAULT  all  --  0.0.0.0/0            0.0.0.0/0            state NEW
WEAVE-NPC-INGRESS  all  --  0.0.0.0/0            0.0.0.0/0            state NEW
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ! match-set weave-local-pods dst

Chain WEAVE-NPC-DEFAULT (1 references)
target     prot opt source               destination
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            match-set weave-E.1.0W^NGSp]0_t5WwH/]gX@L dst /* DefaultAllow isolation for namespace: default */
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            match-set weave-?b%zl9GIe0AET1(QI^7NWe*fO dst /* DefaultAllow isolation for namespace: kube-system */
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            match-set weave-0EHD/vdN#O4]V?o4Tx7kS;APH dst /* DefaultAllow isolation for namespace: kube-public */

Chain WEAVE-NPC-INGRESS (1 references)
target     prot opt source               destination```
```

### Part 0.3 - Leveraging Kubernetes and Cloud Native Microservices
:raised_hands: Also, [Learn more about Imperative vs Declarative deployments](https://medium.com/bitnami-perspectives/imperative-declarative-and-a-few-kubectl-tricks-9d6deabdde)

```console
$ kubectl create namespace microservices
$ kubectl -n=microservices apply -f https://raw.githubusercontent.com/kubernauts/dok-example-us/master/stock-gen/app.yaml 
$ kubectl -n=microservices apply -f https://raw.githubusercontent.com/kubernauts/dok-example-us/master/stock-con/app.yaml
```

After deployment, Let's see where we are...
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
> Back on the host, you can also issues any of these commands by, for instance:
>

```console
host@macbook-pro:/Users/ramene# KUBECONFIG=kube.config kubectl get no,po,svc,deployments --all-namespaces -o wide
```

Now, let's expose the service outside the cluster

```console
root@master:/home/vagrant# kubectl get -n microservices po --selector=app=stock-con \
        -o=custom-columns=:metadata.name --no-headers | \
        xargs -IPOD kubectl -n microservices port-forward POD 9898:9898
```

Let's get our `cluster-info` again for reference sake

```console
root@master:/home/vagrant# kubectl cluster-info
Kubernetes master is running at https://192.168.26.10:6443
KubeDNS is running at https://192.168.26.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Wrapping up.

```console
root@master:/home/vagrant# kubectl delete ns microservices
```

_In Part 2, we'll go over using `helm` to deploy complete application stacks and build pipelines., using our cluster._
