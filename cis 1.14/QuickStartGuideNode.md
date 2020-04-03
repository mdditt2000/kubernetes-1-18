# Kubernetes 1.18 and Container Ingress Controller nodeport Quick Start Guide

This page is created to document K8S 1.16 with integration of CIS and BIG-IP. Please contact me at m.dittmer@f5.com if you have any questions

# Note

Environment parameters

* K8S 1.18 - one master and two worker nodes
* CIS 1.14
* AS3: 3.17.1
* BIG-IP 14.1.2

# Kubernetes 1.18 Install

K8S is installed on RHEL 7.5 on ESXi

* ks8-1-18-master  
* ks8-1-18-node1
* ks8-1-18-node2 (standalone deployment)

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIG-IP. Follow the link to install AS3
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

**Note** When using nodeport, pool members represent the kube-proxy service on the node. BIG-IP needs a local route to the nodes. There is no need for VXLAN tunnels, or Calico. BIG-IP can dynamically ARP for the Kube-proxy running on node

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

Configuration options available in the CIS controller using user-defined configmap
```
args: 
     - "--bigip-username=$(BIGIP_USERNAME)"
     - "--bigip-password=$(BIGIP_PASSWORD)"
     - "--bigip-url=192.168.200.92"
     - "--bigip-partition=k8s"
     - "--namespace=default"
     - "--pool-member-type=nodeport"    ----- As per code it will process as nodeport
     - "--log-level=DEBUG"
     - "--insecure=true"
     - "--manage-ingress=false"
     - "--manage-routes=false"
     - "--agent=as3"
     - "--as3-validation=true"
```
## Create Service for type: NodePort
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
Please look for example files in my repo to clone

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

```
#create kubernetes bigip container connecter, authentication and RBAC
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=f5PME123
kubectl create serviceaccount k8s-bigip-ctlr -n kube-system
kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr
kubectl create -f f5-cluster-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```