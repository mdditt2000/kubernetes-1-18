# Kubernetes 1.16 and Container Ingress Controller Quick Start Guide

This page is created to document K8S 1.16 with integration of CIS and BIGIP. Please contact me at m.dittmer@f5.com if you have any questions

# Note

Environment parameters

* K8S 1.12 - one master and two worker nodes
* CIS 1.12 Beta
* AS3: 3.16.5
* BIG-IP 14.1

# Kubernetes 1.16 Install

K8S is installed on RHEL 7.5 on ESXi

* ks8-1-16-master  
* ks8-1-16-node1
* ks8-1-16-node2

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIGIP. Follow the link to install AS3
 
* Install AS3 on BIGIP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

## Deploy flannel for Kubernetes

Add the BIGIP device to the flannel overlay network. Find the flannel Annotations

```
flannel.alpha.coreos.com/backend-data: {"VtepMAC":"0a:de:a8:5e:00:3f"}
flannel.alpha.coreos.com/backend-type: vxlan
flannel.alpha.coreos.com/kube-subnet-manager: true
flannel.alpha.coreos.com/public-ip: 192.168.200.86
```
## Create a “dummy” Kubernetes Node for the BIGIP device

Include all of the flannel Annotations. Define the backend-data and public-ip Annotations with data from the BIG-IP VXLAN:

```
apiVersion: v1
kind: Node
metadata:
  name: bigip1
  annotations:
    #Replace MAC with your BIGIP Flannel VXLAN Tunnel MAC
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"0a:de:a8:5e:00:3f"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    #Replace IP with Self-IP for your deployment
    flannel.alpha.coreos.com/public-ip: "192.168.200.91"
spec:
  #Replace Subnet with your BIGIP Flannel Subnet
  podCIDR: "10.244.20.0/24
```
## Create a BIG-IP VXLAN tunnel

## create net tunnels vxlan new partition on your BIGIP system
```
tmsh create auth partition k8s
tmsh create net tunnels vxlan fl-vxlan port 8472 flooding-type none
tmsh create net tunnels tunnel fl-vxlan key 1 profile fl-vxlan local-address 192.168.200.91
tmsh create net self 10.244.20.91 address 10.244.20.91/255.255.0.0 allow-service none vlan fl-vxlan
```
## Create CIS Controller, BIGIP credentials and RBAC Authentication

Configuration options available in the CIS controller

```
args: [    
        # See the k8s-bigip-ctlr documentation for information about
        # all config options
        # https://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/latest
        "--bigip-username=$(BIGIP_USERNAME)",
        "--bigip-password=$(BIGIP_PASSWORD)",
        # Replace with the IP address or hostname of your BIG-IP device
        "--bigip-url=192.168.200.91",
        "--bigip-partition=k8s",
        "--namespace=default",
        "--pool-member-type=cluster",
        "--flannel-name=fl-vxlan",
        # Logging level
        "--log-level=DEBUG",
        "--log-response-body",
        AS3 override functionality
        "--override-as3-declaration=default/f5-as3-configmap>",
        # Self-signed cert
        "--insecure=true",
        "--agent=as3",
       ]
```
```
oc create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=f5PME123
oc create serviceaccount bigip-ctlr -n kube-system
oc create -f f5-kctlr-openshift-clusterrole.yaml
oc create -f f5-k8s-bigip-ctlr-openshift.yaml
oc adm policy add-cluster-role-to-user cluster-admin -z bigip-ctlr -n kube-system
```
## Delete kubernetes bigip container connecter, authentication and RBAC
```
oc delete serviceaccount bigip-ctlr -n kube-system
oc delete -f f5-kctlr-openshift-clusterrole.yaml
oc delete -f f5-k8s-bigip-ctlr-openshift.yaml
oc delete secret bigip-login -n kube-system
```
## Create container f5-demo-app-route
```
oc create -f f5-demo-app-route-deployment.yaml -n f5demo
oc create -f f5-demo-app-route-service.yaml -n f5demo
oc create -f f5-demo-app-route-basic.yaml -n f5demo
```
Please look for example files in my repo

## Delete container f5-demo-app-route
```
oc delete -f f5-demo-app-route-deployment.yaml -n f5demo
oc delete -f f5-demo-app-route-service.yaml -n f5demo
oc delete -f f5-demo-app-route-basic.yaml -n f5demo
``` 
## Enable logging for AS3
```
oc get pod -n kube-system
oc log -f f5-server-### -n kube-system | grep -i 'as3'
```