# Migration and Upgrade Guide

This page is created to provide guidance on how to migrate and upgrade the following:

* migration from iApp ConfigMap using CCCL to AS3 ConfigMap
* upgrading from CIS 1.14 using CCCL to AS3
* removing of _AS3 partition
* CIS and AS3 support schema versions

# Migration from iApp ConfigMap using CCCL to AS3 ConfigMap
When migrating from iApp ConfigMap using CCCL to AS3 ConfigMap there are some differences and gotchas. In CIS, configuration of ConfigMap are agent-specific, meaning that configuration in ConfigMap differs from agent to agent. CIS v2.0 the default agent changed from CCCL to AS3. Therefore when migrating from iApp ConfigMap to AS3 ConfigMap using CIS 2.x you do not need to specify the agent as a CIS argument. However if upgrade from CIS 1.x to CIS 2.x and wanting to continue using iApp ConfigMap, specify the CCCL agent type as –agent=CCCL.

Application Services 3 Extension (referred to as AS3 Extension or more often simply AS3) is a flexible, low-overhead mechanism for managing application-specific configurations on a BIG-IP system. The AS3 Extension is required to be installed on BIG-IP. Follow the following document to install AS3 Extension on BIG-IP:

## Install AS3 on BIGIP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

Another difference between iApp ConfigMap using CCCL to AS3 ConfigMap is the concept of tenants. iApp ConfigMap using CCCL the BIG-IP objects are created in BIG-IP using the CIS base partition. When using a AS3 ConfigMap the objects are create in the tenants defined in the JSON declaration. AS3 will create the tenant if the tenant does not exist on BIG-IP. In the example below from the AS3 ConfigMap the tenant is defined as k8s_app. The CIS base tenant cannot be used for the AS3 ConfigMap as in the iApp ConfigMap using CCCL.

Also since AS3 is declarative and declared to BIG-IP by CIS as a single JSON, all the applications need to be added to JSON declaration. CIS unlike Route, Ingress and CRDs wont maintain the state of the AS3 json from the AS3 ConfigMap. In the example below, one application myService_f5_http has been defined. When defining the second application it would can defined in the same tenant as myService_f5_https for example. It is best practice to define your AS3 ConfigMap tenant name the same as the K8S namespace.

Example of ConfigMap with iApp

    kind: ConfigMap
    apiVersion: v1
    metadata:
    name: k8s.http
    namespace: default
    labels:
        f5type: virtual-server
    data:
    # See the f5-schema table for schema-controller compatibility
    # https://clouddocs.f5.com/containers/latest/releases_and_versioning.html#f5-schema
    schema: "f5schemadb://bigip-virtual-server_v0.1.7.json"
    data: |
        {
        "virtualServer": {
            "backend": {
            "serviceName": "f5-hello-world",
            "servicePort": 8080
            },
            "frontend": {
            "partition": "k8s",
            "iapp": "/k8s/f5.http_custom",
            "iappPoolMemberTable": {
                "name": "pool__members",
                "columns": [
                    {"name": "addr", "kind": "IPAddress"},
                    {"name": "port", "kind": "Port"},
                    {"name": "connection_limit", "value": "0"}
                ]
            },
            "iappOptions": {
                "description": "myService_f5.http iApp"
            },
            "iappVariables": {
                "monitor__monitor": "/#create_new#",
                "monitor__response": "none",
                "monitor__uri": "/v1/welcome",
                "net__client_mode": "lan",
                "net__server_mode": "lan",
                "pool__addr": "172.17.32.172",
                "pool__pool_to_use": "/#create_new#",
                "pool__port": "80",
                "pool__hosts": "1.1.1.1"
            }
            }
        }
        }

Example of AS3 ConfigMap with AS3 for tenant k8s_app

    kind: ConfigMap
    apiVersion: v1
    metadata:
    name: k8s.http
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
                "schemaVersion": "3.20.0",
                "id": "urn:uuid:33045210-3ab8-4636-9b2a-c98d22ab915d",
                "label": "http",
                "remark": "Simple HTTP application",
                "k8s_app": {
                    "class": "Tenant",
                    "myService_f5_http": {
                        "class": "Application",
                        "template": "generic",
                        "myService_f5_http": {
                            "class": "Service_HTTP",
                            "virtualAddresses": [
                                "10.192.75.111"
                            ],
                            "virtualPort": 8080,
                            "pool": "pool__members"
                        },
                        "pool__members": {
                            "class": "Pool",
                            "monitors": [
                                {
                                    "use": "http_monitor"
                                }
                            ],
                            "members": [
                                {
                                    "servicePort": 80,
                                    "serverAddresses": []
                                }
                            ]
                        },
                        "http_monitor": {
                            "class": "Monitor",
                            "monitorType": "http",
                            "send": "HTTP GET /v1/welcome",
                            "receive": "none",
                            "adaptive": false
                        }
                    }
                }
            }
        }

# Upgrading from CIS 1.14 using CCCL to AS3
# Removing of _AS3 partition
# CIS and AS3 support schema versions