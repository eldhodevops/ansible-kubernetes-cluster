# ansible-kubernetes-cluster
ansible-kubernetes-cluster
# kubernetes-for-ubuntu-cluster


##  Ansible usage
### Step 1 - Install prerequisites
First, install all prerequisites from the previous paragraph. Ensure that a basic Ansible playbook is working properly when using Ubuntu.If you have linux system you can directly install Ansible Playbook.

#### Prepare Ansible Host system
- Setup for ansible host  on Ubuntu system ( It can be use your Laptop , do not use Servers)

```
apt-get update
apt-get -y install software-properties-common
apt-add-repository -y ppa:ansible/ansible
apt-get update
apt-get install -y ansible python-pip 

```
#### Prepare Master and Linux Nodes 

- Create a user with password in the Master,Nodes linux systems to communicate with ansible-linux
- If you required to use with key to login to the linux Master and Nodes use bellow steps
- Generate key without password

```
# Add user
adduser ubuntu
# change the user
su - ubuntu
# Create Key
ssh-keygen
# Copy the Public key to authorized_keys in order to access by Private key
cp ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys
# Change mode
chmod -R 600 ~/.ssh/*
# Change ownership
chown -R ubuntu: ~/.ssh
# Exit to root user and change mode to user only access to the home folder
chmod 700 /home/ubuntu/

#Enable sudo user for the Ubuntu user shown like bellow
ubuntu ALL=(ALL:ALL) ALL
```
- Copy the generated Private key (id_rsa) to the Ansible host system under ~/key directory like ~/key/ubuntu01.pem
- change the permission only read by owner 

```
mkdir ~/key
# copy key id_rsa to Ansible running server
vim ~/key/ubuntu01.pem
chmod 400 ~/key/ubuntu01.pem
```


### Step 2 - Prepare inventory file
A sample inventory file has been provided in [Ansible playbook directory](ansible/inventory). Assuming that you would like to create cluster having the following hosts:

 - Master node: ubuntu01
 - Linux nodes: ubuntu02, ubuntu03
 
 
 I have commented some line to make bellow cluster
 
 - Master node: ubuntu01
 
 **Note: Do not give hostname in CAPITAL LETTERS . Use small letter.**

your inventory should be defined as:

```
[master-ubuntu]
ubuntu01 kubernetes_node_hostname=ubuntu01

[node-ubuntu]
# ubuntu02 kubernetes_node_hostname=ubuntu02_some_hostname
# ubuntu03 kubernetes_node_hostname=ubuntu03_some_hostname

[node:children]
#node-ubuntu

[all-ubuntu:children]
master-ubuntu
# node-ubuntu

```
### Step 3 - Prepare authentication To the Master and Nodes

 All the sample files has been provided in [Ansible playbook group_var directory](ansible/group_vars).
you need to edit with example bellow

- ansible/group_vars/master-ubuntu.ym.

```
# This is the user we created for ansible communication in the master server
ansible_user: ubuntu
# Here you can give ssh password if your server has enabled username password authentication ( You can uncomment it )
#ansible_ssh_pass: ubuntu
# This is the place to give sudo password 
ansible_become_pass: ubuntu
# This is the private key file location to communicate with the server. ( you can comment it if you are using ssh username password enabled server)
ansible_ssh_private_key_file: ~/key/ubuntu01.pem

### Step 3 - Install Kubernetes packages using install-kubernetes.yml playbook
In order to install basic Kubernetes packages on Windows and Linux nodes, run ``install-kubernetes.yml`` playbook:

```
sh installk8s-node.sh
or
ansible-playbook -i inventory install-kubernetes.yml
```
The playbook consists of the following stages:

1. Installation of common modules. This includes various packages required by Kubernetes and recommended configuration, setting up proxy variables, updating OS, changing hostname.
2. Docker installation. 
3. CNI plugins installation. 
4. Kubernetes packages installation (currently 1.10

At this point, the nodes are ready to be initialized by kubeadm as hybrid Kubernetes cluster. 


### Step 4 - Create Kubernetes cluster with kubeadm using create-kubeadm-cluster.yml
In order to initialize the cluster, run ``create-kubeadm-cluster.yml`` playbook:

```
sh create-kubeadm-cluster.sh
Or
ansible-playbook -i inventory create-kubeadm-cluster.yml --tags=init
```
Filtering by ``init`` tag omits installation of Kubernetes packages which have been already installed in Step 3.
Installation consists of the following stages:
##### Master initialization
1. Master node is initialized using kubeadm.
2. Cluster join token is generated using kubeadm for other nodes.
3. Old Flannel routes/networks are deleted, if present.
4. Kube config is copied to current user's HOME.
5. [RBAC role and role binding](ansible/roles/ubuntu/kubernetes-master/files/kube-proxy-node-rbac.yml) for standalone kube-proxy service is applied. It is required for Windows, which does not host kube-proxy as Kubernetes pod, i.e. it is hosted as a traditional system service.
6. An additional node selector for kube-proxy is applied. Node selector ensures that kube-proxy daemonset is only deployed to Linux nodes. On Windows it is not supported yet, hence the standalone system service.
7. Flannel network with host-gw backend is installed. Node selector ``beta.kubernetes.io/os: linux`` has been added in order to prevent from deploying on Windows nodes, where Flannel is being handled independently.
##### Ubuntu nodes initialization
1. Node joins cluster using kubeadm and token generated on master.
2. Old Flannel routes/networks are deleted, if present.
3. Kubelet service is started.
```
ansible-playbook -i inventory reset-kubeadm-cluster.yml
```
And in case you would like to tear down the cluster completely, including the master:

```
ansible-playbook -i inventory reset-kubeadm-cluster.yml --extra-vars="kubernetes_reset_master=True"
```
This playbook performs draining and deleting of the nodes on master node and eventually it resets the node using kubeadm, so that it is available for joining via kubeadm again.

===============================================================================================================================



