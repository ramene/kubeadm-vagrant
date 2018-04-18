# kubeadm-vagrant
Setup Kubernetes Cluster with Kubeadm and Vagrant

Introduction

With reference to steps listed at [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) for setting up the Kubernetes cluster with kubeadm.  Here I make every attemp outlining simple steps to setup your kubernetes cluster with more control on vagrant based virtual machines.

### kubectl In Action

> Kubernetes comes with `kubectl`, a CLI allowing you to interact with your cluster. It supports operations ranging from configuration to managing workloads and services to handle access control to administrative tasks such as node maintenance. In this talk you'll learn everything you need to know getting the most out of `kubectl` (and beyond).

Installation

- Clone the kubeadm-vagrant repo

```git clone https://github.com/ramene/kubeadm-vagrant ```

- Choose your distribution of choice from CentOS/Ubuntu and move to the specific directory.
- Configure the cluster parameters in Vagrantfile. Refer below for details of configuration options.

``` vi Vagrantfile ```

- Hostmanager may be needed

```sh
There are errors in the configuration of this machine. Please fix
the following errors and try again:

Vagrant:
* Unknown configuration section 'hostmanager'.
```

``` vagrant plugin install vagrant-hostmanager ```

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

- Copy kubeconfig to your machine

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


Got some usage examples?

```
$ KUBECONFIG=kube.config kubectl run -h | tail -n+$(kubectl run -h | grep -n Example | grep -Eo '^[^:]+') | head -n $(kubectl run -h | grep -n Options | grep -Eo '^[^:]+')
```

Alternatively, if you'd like to merge your kube.config with ~/.kube/config do the folowing:
KUBECONFIG=~/.kube/config:./kube.config kubectl config view --flatten --> config
cp ~/.kube/config ~/.kube/config_backup
cp config ~/.kube/config
