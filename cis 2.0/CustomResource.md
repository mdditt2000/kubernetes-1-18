# Kubernetes 1.18 and Container Ingress Controller using Custom Resource Definitions 

This page is created to document CIS 2.0 and BIG-IP using CRD Alpha.  

## What are CRDs? Custom resources are extensions of the Kubernetes API. 

* A resource is an endpoint in the Kubernetes API that stores a collection of API objects of a certain kind; for example, the built-in pods resource contains a collection of Pod objects.
* A custom resource is an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation. It represents a customization of a particular Kubernetes installation. However, many core Kubernetes functions are now built using custom resources, making Kubernetes more modular.
*  Custom resources can appear and disappear in a running cluster through dynamic registration, and cluster admins can update custom resources independently of the cluster itself. Once a custom resource is installed, users can create and access its objects using kubectl, just as they do for built-in resources like Pods.

## How F5 CRDs Custom Controller Works

* Controllers registers to the kubernetes client-go using informers to retrieve service, endpoint, virtual server and node changes
* Resource Queue holds the resources to be processed
* Virtual Server is the primary citizen.  Any changes in Service, Endpoint, Node will indirectly affect Virtual Server
* Worker fetches the affected Virtual Servers from Resource Queue to process them

![Image of clusterIP](https://github.com/mdditt2000/kubernetes-1-18/blob/master/cis%202.0/diagrams/2020-04-23_13-00-46.png)

## Environment parameters

* K8S 1.18 - one master and two worker nodes
* CIS 2.0 beta image
* AS3: 3.18
* BIG-IP 14.1.2 (standalone deployment)

## Kubernetes 1.18 Install

K8S is installed on RHEL 7.5 on ESXi

* ks8-1-18-master  
* ks8-1-18-node1
* ks8-1-18-node2

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIG-IP. Follow the link to install AS3. AS3.18 is required for CIS 2.0
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

## Custom Resource to BIG-IP

* Validate the Virtual Server
* Process each valid Virtual Server to populate to a common structure that holds all the Virtual Servers
* Update Pool Members and L7 LTM policy actions for each Virtual Server
* Controller with help of  Vxlan Manager to create only NET configuration in CCCL as AS3 cannot update L2-L3
* All the overhead of LTM CCCL processing is removed
* LTM(from AS3) and NET(from CCCL) will be created in CIS Managed Partition, which is created by User

![Image of clusterIP](https://github.com/mdditt2000/kubernetes-1-18/blob/master/cis%202.0/diagrams/2020-04-23_13-00-46.png)

**BIG-IP partition**

When using **agent=as3**, CIS will manage L4-L7 and L2-L3 with different partitions CIS would append the configured bigip-partition <partition>_AS3 suffix to partition for L4-L7 operation and use only bigip-partition <partition> for L2-L3 operations

When using user-defined configmap with **agent=as3**, CIS will also manage L4-L7 and L2-L3 with different partitions. CIS would use the <tenant> from the AS3 declaration for L4-L7 operation and use only bigip-partition <partition> for L2-L3 operations

## Create a “dummy” Kubernetes Node for the BIG-IP device

Include all of the flannel Annotations. Define the backend-data and public-ip Annotations with data from the BIG-IP VXLAN:

```
apiVersion: v1
kind: Node
metadata:
  name: bigip1
  annotations:
    #Replace MAC with your BIGIP Flannel VXLAN Tunnel MAC
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"00:50:56:bb:70:8b"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    #Replace IP with Self-IP for your deployment
    flannel.alpha.coreos.com/public-ip: "192.168.200.92"
spec:
  #Replace Subnet with your BIGIP Flannel Subnet
  podCIDR: "10.244.20.0/24
```

**Manage resources**

Specify what resources are configured with the following three options. This guide is using user-defined configmap

* manageRoutes = "manage-routes", false, specify whether or not to manage Route resources
* manageIngress = "manage-ingress", false, specify whether or not to manage Ingress resources
* manageConfigMaps = "manage-configmaps", true, specify whether or not to manage ConfigMap resources

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

Configuration options available in the CIS controller using user-defined configmap
```
args: 
     - "--bigip-username=$(BIGIP_USERNAME)"
     - "--bigip-password=$(BIGIP_PASSWORD)"
     - "--bigip-url=192.168.200.92"
     - "--bigip-partition=k8s"
     - "--namespace=default"
     - "--pool-member-type=cluster" - As per code it will process as clusterIP
     - "--flannel-name=fl-vxlan"
     - "--log-level=DEBUG"
     - "--insecure=true"
     - "--manage-ingress=false"
     - "--manage-routes=false"
     - "--manage-configmaps=true"
     - "--agent=as3"
     - "--as3-validation=true"
```

## BIG-IP credentials and RBAC Authentication

```
#create kubernetes bigip container connecter, authentication and RBAC
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=f5PME123
kubectl create serviceaccount k8s-bigip-ctlr -n kube-system
kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr
kubectl create -f f5-cluster-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```
Please use the following example files in my repo below:

* CIS deployment [CIS deployment repo](https://github.com/mdditt2000/kubernetes-1-18/tree/master/cis%201.14/big-ip-92)