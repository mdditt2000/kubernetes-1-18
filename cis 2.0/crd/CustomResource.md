# Kubernetes 1.18 and Container Ingress Controller using Custom Resource Definitions 

This page is created to document CIS 2.0 and BIG-IP using CRD Alpha.  

## What are CRDs? Custom resources are extensions of the Kubernetes API. 

* A resource is an endpoint in the Kubernetes API that stores a collection of API objects of a certain kind; for example, the built-in pods resource contains a collection of Pod objects.
* A custom resource is an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation. It represents a customization of a particular Kubernetes installation. However, many core Kubernetes functions are now built using custom resources, making Kubernetes more modular.
*  Custom resources can appear and disappear in a running cluster through dynamic registration, and cluster admins can update custom resources independently of the cluster itself. Once a custom resource is installed, users can create and access its objects using kubectl, just as they do for built-in resources like Pods.

## CIS CRD Schema Validation using OpenAPI

CIS using the following schema for CRDs

![Image of Schema](https://github.com/mdditt2000/kubernetes-1-18/blob/master/cis%202.0/crd/diagrams/2020-04-23_13-33-54.png)

## How F5 CRDs Custom Controller Works

* Controllers registers to the kubernetes client-go using informers to retrieve service, endpoint, virtual server and node changes
* Resource Queue holds the resources to be processed
* Virtual Server is the primary citizen.  Any changes in Service, Endpoint, Node will indirectly affect Virtual Server
* Worker fetches the affected Virtual Servers from Resource Queue to process them

![Image of CRDs](https://github.com/mdditt2000/kubernetes-1-18/blob/master/cis%202.0/crd/diagrams/2020-04-23_13-00-46.png)

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

![Image of CRDs](https://github.com/mdditt2000/kubernetes-1-18/blob/master/cis%202.0/crd/diagrams/2020-04-23_13-17-43.png)

**BIG-IP partition**

Single Partition (Back to Awesome). Partition specified in the deployment manifest is what will be created and used on BIG-IP

```
- "--bigip-partition=k8s"
```

**Manage resources**

Specify what resources are configured with the following three options. This guide is using customer resource defintions customResourceMode

* manageRoutes = "manage-routes", false, specify whether or not to manage Route resources
* manageIngress = "manage-ingress", false, specify whether or not to manage Ingress resources
* manageConfigMaps = "manage-configmaps", true, specify whether or not to manage ConfigMap resources
* customResourceMode = "custom-resource-mode", true, specify whether or not to manage controller processes only F5 Custom Resources

**CRD Alpha**

Image: chandrajakkidi/k8s-bigip-ctlr:2.0.0
 
Supports the following features:
* Supports Custom type: VirtualServer
* Responds to changes in VirtualServer resources
* Responds to changes in Services, Endpoints and Nodes
* Uses only one partition in BIG-IP which is created by user
 
Observations to be made:

* No logs related to App Manager
* No CCCL logs related to LTM
* LTM and Net resources in the same partition that is configured
* Traffic validation

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

Configuration options available in the CIS controller using custom resource mode. 

**note** that a deployment parameter needs to be given as “--custom-resource-mode=true” which deploying CIS.
```
args: 
          args: 
            - "--bigip-username=$(BIGIP_USERNAME)"
            - "--bigip-password=$(BIGIP_PASSWORD)"
            - "--bigip-url=192.168.200.92"
            - "--bigip-partition=k8s"
            - "--namespace=default"
            - "--pool-member-type=cluster"
            - "--flannel-name=fl-vxlan"
            - "--log-level=DEBUG"
            - "--insecure=true"
            - "--manage-ingress=false"
            - "--manage-routes=false"
            - "--manage-configmaps=false"
            - "--custom-resource-mode=true"
            - "--http-listen-address=0.0.0.0:8080"
            - "--as3-validation=true"
            - "--log-as3-response=true"
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

## Create the CIS CRD schema

To use CRD with CIS create the schema before creating any CRDs
```
[root@k8s-1-18-master crd-examples]# kubectl create -f customresourcedefinition.yaml
customresourcedefinition.apiextensions.k8s.io/virtualservers.cis.f5.com created
```
This release CRDs are alpha and therefore the ojects are limeted. The following objects are exposed by the schema
```
virtualServerAddress:
pools:
  - path: /
    service: name
    servicePort: ##

```

## Create the CIS CRD
Create a custom resource defintion for a simple application using the following example
```
apiVersion: "cis.f5.com/v1"
kind: VirtualServer
metadata:
  name: f5-hello-world
  labels:
    f5cr: "true"
spec:
  virtualServerAddress: "10.192.75.106"
  pools:
  - path: /
    service: f5-hello-world
    servicePort: 8080
```
Use the kubectl command to create the CRD 
```
[root@k8s-1-18-master crd-examples]# kubectl create -f example-single-pool-virtual.yaml
virtualserver.cis.f5.com/f5-hello-world created
```
Logs dispaly CRD created on BIG-IP using AS3 API
```
2020/05/19 23:22:17 [DEBUG] [AS3] PostManager Accepted the configuration
2020/05/19 23:22:17 [DEBUG] [AS3] posting request to https://192.168.200.92/mgmt/shared/appsvcs/declare/
2020/05/19 23:22:20 [DEBUG] [AS3] Response from BIG-IP: code: 200 --- tenant:k8s --- message: success
```

Please use the following example files in my repo below:

* CIS example [CRD example repo](https://github.com/mdditt2000/kubernetes-1-18/tree/master/cis%202.0/crd/big-ip-92-cluster/crd-examples)