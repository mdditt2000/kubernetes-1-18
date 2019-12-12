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

Add the BIGIP device to the flannel overlay network. Find the VTEP MAC address

```
root@(big-ip-ve1-pme)(cfg-sync Standalone)(Active)(/Common)(tmos)# show net tunnels tunnel fl-vxlan all-properties

-------------------------------------------------
Net::Tunnel: fl-vxlan
-------------------------------------------------
MAC Address                     00:50:56:bb:e5:56
Interface Name                           fl-vxlan

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
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"00:50:56:bb:e5:56"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    #Replace IP with Self-IP for your deployment
    flannel.alpha.coreos.com/public-ip: "192.168.200.91"
spec:
  #Replace Subnet with your BIGIP Flannel Subnet
  podCIDR: "10.244.20.0/24
```
## Create net tunnels vxlan new partition on your BIGIP system
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
        "--log-as3-response=true",
        AS3 override functionality
        "--override-as3-declaration=default/f5-as3-configmap",
        # Self-signed cert
        "--insecure=true",
        "--agent=as3",
       ]
```
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
kubectl create -f f5-cluster-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```
## Delete kubernetes bigip container connecter, authentication and RBAC
```
#delete kubernetes bigip container connecter, authentication and RBAC 
kubectl delete node bigip1
kubectl delete deployment k8s-bigip-ctlr-deployment -n kube-system
kubectl delete clusterrolebinding k8s-bigip-ctlr-clusteradmin
kubectl delete serviceaccount k8s-bigip-ctlr -n kube-system
kubectl delete secret bigip-login -n kube-system
```
## Create ingress and configmap
```
kubectl create -f f5-as3-configmap.yaml
kubectl create -f f5-k8s-ingress.yaml
```
Please look for example files in my repo

## Delete ingress
```
kubectl delete -f f5-k8s-ingress.yaml
``` 
## Enable logging for AS3
```
oc get pod -n kube-system
oc log -f f5-server-### -n kube-system | grep -i 'as3'
```
## CIS supports clientssl, serverssl and waf annotation in ingress object

CIS supports clientssl, serverssl and waf annotation in ingress object using CIS override config-map feature

### 1.clientssl support

#### Steps to support clientssl in ingress:

1. Create kubernetes secret.

2. Attach kubernetes secret to your ingress.

Example: 1
```
apiVersion: v1
kind: Secret
metadata:
  name: secret-1
type: Opaque
data:
  tls.crt: <-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE----->
  tls.key: <-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY----->

---

apiVersion: v1
kind: Secret
metadata:
  name: secret-2
type: Opaque
data:
  tls.crt: <-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE----->
  tls.key: <-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE----->

---

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    virtual-server.f5.com/ip: controller-default
  name: client-secret
spec:
  backend:
    serviceName: svc
    servicePort: 80
  tls:
  - secretName: secret-1
  - secretName: secret-2
  . . . 
  . . . 
  - secretName: secret-N 
  # Multiple secrets can be provided as per the Ingress requirement.
```

Example: 2

Using BIG-IP SSL profile attached to TLS

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingressTLS
  namespace: default
  annotations:
  # See the k8s-bigip-ctlr documentation for information about
  # all Ingress Annotations
  # https://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/latest/#supported-ingress-annotations
    virtual-server.f5.com/ip: "1.2.3.4"
    ingress.kubernetes.io/ssl-redirect: "true"
    ingress.kubernetes.io/allow-http: "false"
spec:
  tls:
    # Provide the name of the BIG-IP SSL profile you want to use.
    - secretName: /Common/clientssl
  backend:
    # Provide the name of a single Kubernetes Service you want to expose to external
    # traffic using TLS
    serviceName: myService
    servicePort: 443
```

### 2. serverssl support

Ingress support serverssl using annotations.

Annotation: ingress.kubernetes.io/serverssl

Example:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingressTLS
  namespace: default
  annotations:
  # See the k8s-bigip-ctlr documentation for information about
  # all Ingress Annotations
  # https://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/latest/#supported-ingress-annotations
    virtual-server.f5.com/ip: "1.2.3.4"
    virtual-server.f5.com/partition: "k8s"
    ingress.kubernetes.io/ssl-redirect: "true"
    ingress.kubernetes.io/allow-http: "false"
    # Provide the name of the BIG-IP SSL profile you want to use.
	ingress.kubernetes.io/serverssl: "/Common/serverssl"
spec:
  backend:
    # Provide the name of a single Kubernetes Service you want to expose to external
    serviceName: myService
    servicePort: 443
```

### 3. waf support

Ingress support web application firewall (waf) using configmap override functionality.

How to use override functionality doc:

       https://clouddocs.f5.com/
#### Steps to use use override for WAF.

1. Create override configmap with WAF declaration that need to be incorporated with VS.

2. Apply your override configmap.


Example:

Below declaration is actual AS3 declaration generated by CIS without WAF.

```
{
  "$schema": "https://raw.githubusercontent.com/F5Networks/f5-appsvcs-extension/master/schema/latest/as3-schema-3.11.0-3.json",
  "class": "AS3",
  "declaration": {
    "class": "ADC",
    "id": "urn:uuid:B97DFADF-9F0D-4F6C-8D66-E9B52E593694",
    "label": "CIS Declaration",
    "remark": "Auto-generated by CIS",
    "schemaVersion": "3.11.0",
    "test_AS3": {
      "Shared": {
        "class": "Application",
        "ingress_172_16_3_23_443": {
          "layer4": "tcp",
          "source": "0.0.0.0/0",
          "translateServerAddress": true,
          "translateServerPort": true,
          "class": "Service_HTTPS",
          "profileHTTP": {
            "bigip": "/Common/http"
          },
          "profileTCP": {
            "bigip": "/Common/tcp"
          },
          "virtualAddresses": [
            "172.16.3.23"
          ],
          "virtualPort": 443,
          "snat": "auto",
          "clientTLS": {
            "bigip": "/Common/serverssl"
          },
          "serverTLS": {
            "bigip": "/Common/clientssl"
          },
          "redirect80": false,
          "pool": "/test_AS3/Shared/ingress_default_svc"
        },
        "ingress_172_16_3_23_80": {
          "layer4": "tcp",
          "source": "0.0.0.0/0",
          "translateServerAddress": true,
          "translateServerPort": true,
          "class": "Service_HTTP",
          "profileHTTP": {
            "bigip": "/Common/http"
          },
          "profileTCP": {
            "bigip": "/Common/tcp"
          },
          "virtualAddresses": [
            "172.16.3.23"
          ],
          "virtualPort": 80,
          "snat": "auto",
          "pool": "/test_AS3/Shared/ingress_default_svc"
        },
        "ingress_default_svc": {
          "class": "Pool",
          "loadBalancingMode": "round-robin",
          "members": [
            {
              "addressDiscovery": "static",
              "serverAddresses": [
                "172.16.1.12"
              ],
              "servicePort": 32289
            },
            {
              "addressDiscovery": "static",
              "serverAddresses": [
                "172.16.1.10"
              ],
              "servicePort": 32289
            }
          ]
        },
        "template": "shared"
      },
      "class": "Tenant"
    }
  }
}
```

Let apply WAF to Virtual Server "ingress_172_16_3_23_443"

Create a override configmap with WAF declaration:

Eaxmple:

This example attaches a Web Application Firewall (WAF) security policy to virtual server.

### Note: Need to follow the same AS3 hierarchy to add new declaration.

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: example-override
  namespace: default
data:
  template: |
    {
        "declaration": {
            "test_AS3": {
                "Shared": {
                    "ingress_172_16_3_23_443": {
                        "policyWAF": {
                            "use": "My_ASM_Policy"
                         }
                    },
                    "My_ASM_Policy": {
                        "class": "WAF_Policy",
                        "file": "/Common/my_asm_waf",
                        "ignoreChanges": true
                    },
                }
            }
        }
    }
```

CIS generates final AS3 declaration with WAF after applying the override config-map:

```
{
  "$schema": "https://raw.githubusercontent.com/F5Networks/f5-appsvcs-extension/master/schema/latest/as3-schema-3.11.0-3.json",
  "class": "AS3",
  "declaration": {
    "class": "ADC",
    "id": "urn:uuid:B97DFADF-9F0D-4F6C-8D66-E9B52E593694",
    "label": "CIS Declaration",
    "remark": "Auto-generated by CIS",
    "schemaVersion": "3.11.0",
    "test_AS3": {
      "Shared": {
        "class": "Application",
        "ingress_172_16_3_23_443": {
          "layer4": "tcp",
          "source": "0.0.0.0/0",
          "translateServerAddress": true,
          "translateServerPort": true,
          "class": "Service_HTTPS",
          "profileHTTP": {
            "bigip": "/Common/http"
          },
          "profileTCP": {
            "bigip": "/Common/tcp"
          },
          "virtualAddresses": [
            "172.16.3.23"
          ],
          "policyWAF": {
            "use": "My_ASM_Policy"
          },
          "virtualPort": 443,
          "snat": "auto",
          "clientTLS": {
            "bigip": "/Common/serverssl"
          },
          "serverTLS": {
            "bigip": "/Common/clientssl"
          },
          "redirect80": false,
          "pool": "/test_AS3/Shared/ingress_default_svc"
        },
        "My_ASM_Policy": {
          "class": "WAF_Policy",
          "file": "/Common/my_asm_waf",
          "ignoreChanges": true
        },
        "ingress_172_16_3_23_80": {
          "layer4": "tcp",
          "source": "0.0.0.0/0",
          "translateServerAddress": true,
          "translateServerPort": true,
          "class": "Service_HTTP",
          "profileHTTP": {
            "bigip": "/Common/http"
          },
          "profileTCP": {
            "bigip": "/Common/tcp"
          },
          "virtualAddresses": [
            "172.16.3.23"
          ],
          "virtualPort": 80,
          "snat": "auto",
          "pool": "/test_AS3/Shared/ingress_default_svc"
        },
        "ingress_default_svc": {
          "class": "Pool",
          "loadBalancingMode": "round-robin",
          "members": [
            {
              "addressDiscovery": "static",
              "serverAddresses": [
                "172.16.1.12"
              ],
              "servicePort": 32289
            },
            {
              "addressDiscovery": "static",
              "serverAddresses": [
                "172.16.1.10"
              ],
              "servicePort": 32289
            }
          ]
        },
        "template": "shared"
      },
      "class": "Tenant"
    }
  }
}
```
