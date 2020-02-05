# Kubernetes FAQ for Container Ingress Controller

## Kubernetes node install with multiple interfaces

**Problem:** CIS takes the node ip information from kube api in cluster mode. It should actually take these details from flannel as CIS creates the fdb entries in BIG-IP using these details. This issue is only seen when the nodes are multiple interfaces. 

**Solution:** Use flannel annotations to get the node ip addresses. Add a new annotation for mac address and public IP. This needs to get added to each node. Please find the required annotations below. MAC address of the vxlan interface of that particular node.

```
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"<MAC>"}'
    flannel.alpha.coreos.com/public-ip: “<IP>”
```
In kubernetes edit the node resource file

Github issue https://github.com/F5Networks/k8s-bigip-ctlr/issues/797

---

## CIS log messages

Information regarding the following error messages and what do they mean

```
2020/01/29 20:35:53 [DEBUG] Using agent as3
2020/01/29 20:35:53 [DEBUG] [AS3] Invalid trusted-certs-cfgmap option provided.
2020/01/29 20:35:53 [DEBUG] [AS3] No certs appended, using only system certs
2020/01/29 20:35:53 [DEBUG] Error while fetching latest as3 schema : Get https://raw.githubusercontent.com/F5Networks/f5-appsvcs-extension/master/schema/latest/as3-schema.json: dial tcp 151.101.92.133:443: connect: no route to host
2020/01/29 20:35:53 [DEBUG] Unable to fetch the latest AS3 schema : validating AS3 schema with as3-schema-3.13.2-1-cis.json
```