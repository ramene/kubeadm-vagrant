# kubeadm-vagrant
Setup Kubernetes Cluster with Kubeadm and Vagrant

Introduction

With reference to steps listed at [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) for setting up the Kubernetes cluster with kubeadm. I have been working on an automation to setup the cluster. The result of it is [kubeadm-vagrant](https://github.com/coolsvap/kubeadm-vagrant), a github project with simple steps to setup your kubernetes cluster with more control on vagrant based virtual machines.

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

N.B.: if you'd like to merge your kube.config with ~/.kube/config do the folowing:
KUBECONFIG=~/.kube/config:./kube.config kubectl config view --flatten --> config
cp ~/.kube/config ~/.kube/config_backup
cp config ~/.kube/config

```
1. ``` BOX_IMAGE ``` is currently default with &quot;coolsvap/centos-k8s&quot; box which is custom box created which can be used for setting up the cluster with basic dependencies for kubernetes node.
2. Set ``` SETUP_MASTER ``` to true if you want to setup the node. This is true by default for spawning a new cluster. You can skip it for adding new minions.
3. Set ``` SETUP_NODES ``` to false by the first run, after the master is created and you run ```kubeadm token create``` on master an update your vagrantfile with the token, then you need to set ``` SETUP_NODES ``` to true.
4. Specify ``` NODE_COUNT ``` as the count of minions in the cluster
5. Specify  the ``` MASTER_IP ``` as static IP which can be referenced for other cluster configurations
6. Specify ``` NODE_IP_NW ``` as the network IP which can be used for assigning dynamic IPs for cluster nodes from the same network as Master
7. Specify custom ``` POD_NW_CIDR ``` of your choice
8. Setting up kubernetes dashboard is still a WIP with ``` K8S_DASHBOARD ``` option.
```
