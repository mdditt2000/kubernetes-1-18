# Kubernetes 1.16 and Container Ingress Controller Quick Start Guide for using type-loadbalance

This page is created to document K8S 1.16 with integration of CIS and BIGIP using kubernetes type-loadbalance 

# Note

Environment parameters

* K8S 1.12 - one master and two worker nodes
* CIS 1.12
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

## Deploy nodeport code base

No vxlan tunnels are needed for nodeport

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
            "--bigip-url=192.168.200.92",
            "--bigip-partition=k8s",
            "--namespace=default",
            "--pool-member-type=nodeport", ----- As per code it will process as nodeport
            # Logging level
            "--log-level=DEBUG",
            "--log-as3-response=true",
            # AS3 override functionality
            #"--override-as3-declaration=default/f5-as3-declaration",
            # Self-signed cert
            "--insecure=true",
            "--agent=as3",
          ]
```
**Note:** As per code it will process as nodeport but the service is configured as type-loadbalance 

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
  type: LoadBalancer
  ```

Please let me know if you require additional information