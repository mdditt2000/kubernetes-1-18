CIS managed partitions <partition_AS3> and <partition> should not be used in ConfigMap as Tenants
Eg:
If CIS is deployed with "--bigip-partition=cis", then <cis_AS3> and <cis> are not supposed to be used as a Tenant in AS3 declaration.
Below is an improper declartion which would not be processed processed by CIS. In my example im using Tenant k8s in AS3 declaration ad shown the example below.
```
apiVersion: v1
data:
  template: |
    {
        "class": "AS3",
        "action": "deploy",
        "persist": true,
        "declaration": {
            "class": "ADC",
            "schemaVersion": "3.10.0",
            "id": "urn:uuid:33045210-3ab8-4636-9b2a-c98d22ab915d",
            "label": "example_http_application_01",
            "remark": "Simple HTTP application with RR pool",
            "test": {
                "class": "Tenant",
                "example_http_application_01": {
                    "class": "Application",
                    "template": "http",
                    "serviceMain": {
                        "class": "Service_HTTP",
                        "virtualAddresses": [
                            "172.16.3.11"
                        ],
                        "pool": "web_pool"
                    },
                    "web_pool": {
                        "class": "Pool",
                        "loadBalancingMode": "predictive-node",
                        "monitors": [
                            "http"
                        ],
                        "members": [
                            {
                                "servicePort": 80,
                                "serverAddresses": [
                                ]
                            }
                        ]
                    }
                }
            }
        }
    }
kind: ConfigMap
metadata:
  creationTimestamp: 2019-10-16T11:29:07Z
  labels:
    as3: "true"
    f5type: virtual-server
  name: example-http-application-01
  namespace: default
  resourceVersion: "9107165"
  selfLink: /api/v1/namespaces/default/configmaps/example-http-application-01
  uid: 2a42f382-f008-11e9-bb5a-fa163e17f50f
  
  ```
  
