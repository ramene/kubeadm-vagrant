# kubeadm-vagrant
Setup Kubernetes Cluster with Kubeadm and Vagrant

Introduction

With reference to steps listed at [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) for setting up the Kubernetes cluster with kubeadm. I have been working on an automation to setup the cluster. The result of it is [kubeadm-vagrant](https://github.com/coolsvap/kubeadm-vagrant), a github project with simple steps to setup your kubernetes cluster with more control on vagrant based virtual machines.

Installation

- Clone the kubeadm-vagrant repo

```git clone https://github.com/arashkaffamanesh/kubeadm-vagrant ```

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
148a37.736fd53655b767b7 --> you need to set this token in vagrantfile
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
```

