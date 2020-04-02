# Kubernetes 1.18 and Container Ingress Controller Deployment Guide for HA

This page is created to document K8S 1.16 with integration of CIS and BIGIP

**Note** We have observed quite a few issues when performed some actual testing of BIGIP HA with K8s using Flannel. Data Traffic through associated external floating IP doesn't work due to replication of FDB configurations from ACTIVE to STANDBY not happening properly. Therefore the only recommend approach for BIGIP HA with K8S is using Calico or Nodeport. This document provides configuration examples and guidance for NodePort based on my testing. 

Here is a document to configuring the BIGIP system to connect the Calico Kubernetes cluster https://support.f5.com/csp/article/K14436300

Environment parameters

* K8S 1.12 - one master and two worker nodes
* CIS 1.12
* AS3: 3.16.5
* 2 BIG-IP 14.1

# Kubernetes 1.16 Install

K8S is installed on RHEL 7.5 on ESXi

* ks8-1-16-master  
* ks8-1-16-node1
* ks8-1-16-node2

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIGIP. Follow the link to install AS3
 
* Install AS3 on BIGIP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

**Note** Since using NodePort there is no need for VXLAN tunnels, additional routing. BIGIP can dynamically ARP for the Kube-proxy running on node. However CIS controller are required. AS3 has a sync on change flag, so when you send a declaration, if something changed, it will sync. There for my testing i used active sync which worked nicely. Also no additional floating self-ip are required. Since the BIGIP is the default gateway I created a self-ip on the internal network and assigned the traffic group 

## Create CIS Controller, BIGIP credentials and RBAC Authentication

Configuration for CIS controller 91
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
            "--pool-member-type=nodeport",
            # Logging level
            "--log-level=DEBUG",
            "--log-as3-response=true",
            # AS3 override functionality
            "--override-as3-declaration=default/f5-as3-declaration",
            # Self-signed cert
            "--insecure=true",
            "--agent=as3",
          ]
```
Configuration for CIS controller 92
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
            "--pool-member-type=nodeport",
            # Logging level
            "--log-level=DEBUG",
            "--log-as3-response=true",
            # AS3 override functionality
            "--override-as3-declaration=default/f5-as3-declaration",
            # Self-signed cert
            "--insecure=true",
            "--agent=as3",
          ]
```
I also test with AS3 override to enable WAF and logging

**Note:** CIS controller is configured with the override-as3-declaration option. This allow the user BIGIP administrator to add global policy, profiles etc to the virtual without having to add additional the need for an annotation. Example below show added WAF and logging. Create this configmap for the configuration to be applied. The configmap, namespace, tenant, AS3 app all need to match. **All the objects need to be defined under the virtual**

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: f5-as3-declaration
  namespace: default
data:
  template: |
    {
        "declaration": {
            "k8s_AS3": {
                "Shared": {
                    "ingress_10_192_75_108_80": {
                        "securityLogProfiles": [
                            {
                                "bigip": "/Common/Log all requests"
                            }
                        ],
                        "policyWAF": {
                            "bigip": "/Common/WAF_Policy"
                        }
                    }
                }
            }
        }
    }
```

## BIGIP credentials and RBAC Authentication

```
#create kubernetes bigip container connecter, authentication and RBAC
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=f5PME123
kubectl create serviceaccount k8s-bigip-ctlr -n kube-system
kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr
kubectl create -f f5-nodeport-deployment-91.yaml

#create kubernetes bigip container connecter, authentication and RBAC
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=f5PME123
kubectl create serviceaccount k8s-bigip-ctlr -n kube-system
kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr
kubectl create -f f5-nodeport-deployment-92.yaml
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