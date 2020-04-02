# Install kubernetes 1.18 cluster using kubeadm
Follow this documentation to set up a Kubernetes cluster on __RHEL 7__ Virtual machines.

This documentation guides you in setting up a cluster with one master node and one worker node.

## Assumptions
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Master|ks8-1-18-master.example.com|192.168.200.93|RHEL 7-5|16G|4|
|Worker|ks8-1-18-node1.example.com|192.168.200.94|RHEL 7-5|16G|4|
|Worker|ks8-1-18-node2.example.com|192.168.200.95|RHEL 7-5|16G|4|

## On both master and worker nodes
Perform all the commands as root user unless otherwise specified
### Pre-requisites
##### Update /etc/hosts
So that we can talk to each of the nodes in the cluster
```
cat >>/etc/hosts<<EOF
192.168.200.93 ks8-1-18-master.example.com ks8-1-18-master
192.168.200.94 ks8-1-18-node1.example.com ks8-1-18-node1
192.168.200.95 ks8-1-18-node2.example.com ks8-1-18-node2
EOF
```
##### Install, enable and start docker service
To install docker on RHEL add the correct OS repository.
```
subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ansible-2.6-rpms"

yum install docker -y
systemctl enable docker
systemctl start docker
```
##### Disable SELinux
```
setenforce 0
sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
```
##### Disable Firewall
```
systemctl disable firewalld
systemctl stop firewalld
```
##### Disable swap
```
sed -i '/swap/d' /etc/fstab
swapoff -a
```
##### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
### Kubernetes Setup
##### Add yum repository
```
cat >>/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
##### Install Kubernetes
```
yum install -y kubeadm kubelet kubectl
```
##### Enable and Start kubelet service
```
systemctl enable kubelet
systemctl start kubelet
```
## On master
##### Initialize Kubernetes Cluster
```
kubeadm init --apiserver-advertise-address=192.168.200.93 --pod-network-cidr=10.244.0.0/16
```
##### Copy kube config
To be able to use kubectl command to connect and interact with the cluster, the user needs kube config file.

In my case, the user account is kube
```
adduser kube
mkdir /home/kube/.kube
cp /etc/kubernetes/admin.conf /home/kube/.kube/config
chown -R kube:kube /home/kube/.kube
```
##### Deploy flannel network
This has to be done as the user in the above step (in my case it is __kube__)
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
##### Cluster join command
```
kubeadm token create --print-join-command
```
## On worker
##### Join the cluster
Use the output from __kubeadm token create__ command in previous step from the master server and run here.

## Verifying the cluster
##### Get Nodes status
```
kubectl get nodes
```
##### Get component status
```
kubectl get cs
```