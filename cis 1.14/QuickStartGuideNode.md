# Kubernetes 1.18 and Container Ingress Controller NodePort Quick Start Guide

This page is created to document K8S 1.18 with integration of CIS and BIG-IP using NodePort configuration. Benefits of NodePort are:

* Works in any environment (no requirement for SDN)
* No persistence/visibility to backend Pod
* Can be deployed for “static” workloads (not ideal)

Similar to vanilla Docker BIG-IP is communicating with a ephemeral port, but in this case the kube-proxy is keeping track of the backend Pod (container). This works well, but the downside is that you have an additional layer of load balancing with the kube-proxy

![Image of NodePort](https://github.com/mdditt2000/kubernetes-1-18/blob/master/cis%201.14/diagrams/2020-04-06_14-57-25.png)

## Environment parameters

* K8S 1.18 - one master and two worker nodes
* CIS 1.14
* AS3: 3.17.1
* BIG-IP 14.1.2 (standalone deployment)

## Kubernetes 1.18 Install

K8S is installed on RHEL 7.5 on ESXi

* ks8-1-18-master  
* ks8-1-18-node1
* ks8-1-18-node2

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIG-IP. Follow the link to install AS3
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

**Networking** 

When using nodeport, pool members represent the kube-proxy service on the node. BIG-IP needs a local route to the nodes. There is no need for VXLAN tunnels, or Calico. BIG-IP can dynamically ARP for the Kube-proxy running on node

**BIG-IP partition**

When using **agent=as3**, CIS will manage L4-L7 and L2-L3 with different partitions CIS would append the configured bigip-partition <partition>_AS3 suffix to partition for L4-L7 operation and use only bigip-partition <partition> for L2-L3 operations

When using user-defined configmap with **agent=as3**, CIS will also manage L4-L7 and L2-L3 with different partitions. CIS would use the <tenant> from the AS3 declaration for L4-L7 operation and use only bigip-partition <partition> for L2-L3 operations

**Manage resources**

Specify what resources are configured with the following three options. This guide is using user-defined configmap

* manageRoutes = "manage-routes", false, specify whether or not to manage Route resources
* manageIngress = "manage-ingress", false, specify whether or not to manage Ingress resources
* manageConfigMaps = "manage-configmaps", true, specify whether or not to manage ConfigMap resources

**Manage node labels**

When using nodeport by default all nodes from the cluster will be added to the pool. In most cases you only want to add the worker nodes and exclude master nodes. To exclude master nodes using the label node-role.kubernetes.io/node parameter

Use the label node <node name> to create a new label for the node. This works in conjunction with the node-label-selector configured in CIS to only add nodes to the pool with the associated <node name>. In this quick start guide CIS will only add the nodes with label worker to the pool. Excluding the master node from the pool 

```
# kubectl label nodes k8s-1-18-node1.example.com node-role.kubernetes.io/f5role=worker
# kubectl label nodes k8s-1-18-node2.example.com node-role.kubernetes.io/f5role=worker
```
Show the node labels
```
# kubectl get nodes
NAME                         STATUS   ROLES    AGE     VERSION
k8s-1-18-master.example.com   Ready    master   7d19h   v1.18.0
k8s-1-18-node1.example.com    Ready    f5role   7d19h   v1.18.0
k8s-1-18-node2.example.com    Ready    f5role   7d19h   v1.18.0
```

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

Configuration options available in the CIS controller using user-defined configmap
```
args: 
     - "--bigip-username=$(BIGIP_USERNAME)"
     - "--bigip-password=$(BIGIP_PASSWORD)"
     - "--bigip-url=192.168.200.92"
     - "--bigip-partition=k8s"
     - "--namespace=default"
     - "--pool-member-type=nodeport" - As per code it will process as nodeport
     - "--log-level=DEBUG"
     - "--insecure=true"
     - "--manage-ingress=false"
     - "--manage-routes=false"
     - "--manage-configmaps=true"
     - "--agent=as3"
     - "--as3-validation=true"
     - "--node-label-selector=f5role=worker"
```
## Create Service for type: NodePort

Specify type: NodePort in the service to configure NodePort in the service

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: f5-hello-world
  name: f5-hello-world
spec:
  ports:
  - name: f5-hello-world
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: f5-hello-world
  type: NodePort
```

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

```
#create kubernetes bigip container connecter, authentication and RBAC
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=f5PME123
kubectl create serviceaccount k8s-bigip-ctlr -n kube-system
kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr
kubectl create -f f5-cluster-deployment.yaml
```
Please use the following example files in my repo below:

* CIS deployment [CIS deployment repo](https://github.com/mdditt2000/kubernetes-1-18/tree/master/cis%201.14/type-nodeport)