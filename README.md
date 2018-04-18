## Running Ethereum and IPFS on K8s with kubeadm-vagrant

> _In [Part 1](https://gist.github.com/ramene/e918aa4664d4c40189bc2119700bf444) we discussed how you would effectively **enable** Kubernetes.  Here, in our continued series of working sessions, we outline the manual process for those that prefer to run a X node Kubernetes Cluster locally with Kubeadm Vagrant (with VirtualBox provider)._
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

```
vagrant ssh master
sudo su
kubectl get pods --all-namespaces
kubeadm token create
148a37.736fd53655b767b7
--> we need to set this token for KUBETOKEN in our Vagrantfile
```

- Spin up the nodes

set SETUP_NODES = true in Vagrantfile

``` vagrant up ```

- Copy kubeconfig to your host machine

```
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


### Everything look good?

As outlined in [Part 1](https://gist.github.com/ramene/e918aa4664d4c40189bc2119700bf444#first-lets-deploy-a-simple-microservice-to-our-local-kubernetes-cluster), let's deploy the [sample microservice](https://github.com/kubernauts/dok-example-us) from [Michael Hausenblas](https://github.com/mhausenblas)

Ok, What else can I do?

```sh
$ KUBECONFIG=kube.config kubectl run -h | tail -n+$(kubectl run -h | grep -n Example | grep -Eo '^[^:]+') | head -n $(kubectl run -h | grep -n Options | grep -Eo '^[^:]+')
```

### Cluster Networking - Advanced

```sh
$ sudo iptables -L -n
```

```console
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

Chain KUBE-SERVICES (1 references)
target     prot opt source               destination
```

This [repository](https://github.com/kubernetes/ingress-nginx) By far the more reasonable and scalable solution is to use an [Ingress controller](https://github.com/kubernetes/ingress-nginx/tree/nginx-0.12.0#what-is-an-ingress-controller) to surface as many Services as you wish, using the same nginx virtual-hosting that you are likely already familiar with using.


_coming in Part 2, we'll cover using Truffle to compile and migrate our smart contracts to our blockchain._
