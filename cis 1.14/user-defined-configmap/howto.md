# Container Ingress Services using AS3 Declarative API
This how-to document demonstrates how CIS take advantage of an declarative API to configure and update a BIG-IP from a kubernetes cluster. This configuration take advantage of cluster mode. In a cluster mode BIG-IP can reach the containers directly. 

## Use Case
Demonstrates the following BIG-IP capabilities 

* HTTP, HTTPS 
* Cookie persistence
* TLS termination
* End to end TLS termination
* Web Application firewall
* Tenant filtering

## Declarative API
The Application Services 3 Extension uses a declarative model, meaning CIS sends a declaration file using a single Rest API call. An AS3 declaration describes the desired configuration of an Application Delivery Controller (ADC) such as F5 BIG-IP in tenant- and application-oriented terms. An AS3 tenant comprises a collection of AS3 applications and related resources responsive to a particular authority (the AS3 tenant becomes a partition on the BIG-IP system). An AS3 application comprises a collection of ADC resources relating to a particular network-based business application or system. AS3 declarations may also include resources shared by Applications in one Tenant or all Tenants as well as auxiliary resources of different kinds.

## Prerequisites for using AS3

Only with CIS 1.12 using agent=AS3. **This section below is still in the works**

**Note:** CIS uses the partition defined in the controller configuration by default to communicate with the F5 BIG-IP when adding static ARPs and forwarding entries for VXLAN. CIS managed partitions **<partition_AS3>** and **<partition>** should not be used in ConfigMap as Tenants. If CIS is deployed with **bigip-partition=cis**, then **<cis_AS3>** and **<cis>** are not supposed to be used as a tenant in AS3 declaration. Below is a proper declaration which would be correctly processed by CIS. Using **<k8s>** for the AS3 tenant in AS3. 

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: f5-as3-declaration
  namespace: default
  labels:
    f5type: virtual-server
    as3: "true"
data:
  template: |
    {
        "class": "AS3",
        "declaration": {
            "class": "ADC",
            "schemaVersion": "3.13.0",
            "id": "urn:uuid:33045210-3ab8-4636-9b2a-c98d22ab915d",
            "label": "http",
            "remark": "A1 Template",
            "k8s": {
                "class": "Tenant",
                "A1": {
                    "class": "Application",
                    "template": "generic",
                    "a1_80_vs": {
                        "class": "Service_HTTP",
                        "remark": "a1",
                        "virtualAddresses": [
                            "10.192.75.101"
                        ],
                        "pool": "web_pool"
                    },
                    "web_pool": {
                        "class": "Pool",
                        "monitors": [
                            "http"
                        ],
                        "members": [
                            {
                                "servicePort": 8080,
                                "serverAddresses": []
                            }
                        ]
                    }
                }
            }
        }
    }
```
**Note:** CIS 1.12 provides the option to use tenant filtering. If you add --filter-tenants=true in the CIS deployment configuration, CIS will all the tenent name to the API call. In this case https://10.192.75.98/mgmt/shared/appsvcs/declare/k8s would be the API call.

## Installing AS3 and handling SSL certificate verification 

* Install the AS3 RPM on the F5 BIG-IP. Following the link https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html
* If the F5 BIG-IP is using un-signed default ssl certificates add **insecure=true** as shown below to the controller deployment yaml file. Example https://github.com/mdditt2000/kubernetes/blob/dev/cis-1-9/big-ip-98/f5-cluster-deployment.yaml
    ```
    args: [
        "--bigip-username=$(BIGIP_USERNAME)",
        "--bigip-password=$(BIGIP_PASSWORD)",
        "--bigip-url=192.168.200.98",
        "--bigip-partition=AS3",
        "--namespace=default",
        "--pool-member-type=cluster",
        "--flannel-name=fl-vxlan",
        "--log-level=INFO",
        "--insecure=true",
        "--filter-tenants=true"
    ```
* Add as3: "true" to any configmap applied so that CIS knows the data fields is AS3 and not legacy container connector input data. Please note that CIS will use gojsonschema to validate the AS3 data. If the declaration doesnt conform with the schema an error will be logged. Example https://github.com/mdditt2000/kubernetes/blob/dev/cis-1-9/blank/f5-as3-configmap.yaml
    ```
    metadata:
    name: f5-hello-world-https
    namespace: default
    labels:
        f5type: virtual-server
        as3: "true"
    ```
* Create and deploy the kuberenetes service discovery lables. CIS can dynamically discover and update load balancing pool members using service discovery. CIS maps each pool definition in the AS3 template to a Kubernetes Service resource using a label. To create this mapping, add the following labels to your Kubernetes Service. Example https://github.com/mdditt2000/kubernetes/blob/dev/cis-1-9/deployment/f5-hello-world-service.yaml
    ```
    labels:
        app: f5-hello-world-end-to-end-ssl
        cis.f5.com/as3-tenant: AS3
        cis.f5.com/as3-app: A5
        cis.f5.com/as3-pool: secure_ssl_waf_pool
    name: f5-hello-world-end-to-end-ssl-waf
    ```
## Using a configmap with AS3
When using CIS with AS3 the behaviors are different The following needs to apply:

* CIS create one JSON declaration 
* Service doesnt matter on the order inside the declaration 
* Deleting a configmap doesnt remove the AS3 declaration. You need to remove the AS3 application first. Update the declaration and kube will post the changes
* To remove the AS3 declaration from BIG-IP usu a blank declaration and displayed in this example: https://github.com/mdditt2000/kubernetes/blob/dev/cis-1-9/blank/f5-as3-configmap.yaml
* Once the declaration is blank the AS3 partition will be removed and you can now delete the configmap
* When adding new services use the kubectl apply command

### AS3 with HTTP application
Deploying a application called A1 for http. Example of the declaration https://github.com/mdditt2000/kubernetes/blob/dev/cis-1-9/A1/f5-as3-configmap.yaml

**Note:** This is the first application to be deployed by kub. This example will deploy a simple http application on BIG-IP
```
[kube@k8s-1-13-master A1]$ kubectl create -f f5-as3-configmap.yaml
configmap/f5-as3-declaration created
```
### AS3 with HTTPs application
Deploy a second appliction called A2 for https. Example of the declaration https://github.com/mdditt2000/kubernetes/blob/dev/cis-1-9/A2/f5-as3-configmap.yaml
```
[kube@k8s-1-13-master A2]$ kubectl get cm
NAME                 DATA   AGE
f5-as3-declaration   1      24m
```
Note the declaration is already created. To deploy a new service simple apply declaration A1 + A2. AS3 running on BIP-IP will detect and implment the changes
```
[kube@k8s-1-13-master A2]$ kubectl apply -f f5-as3-configmap.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
configmap/f5-as3-declaration configured
```