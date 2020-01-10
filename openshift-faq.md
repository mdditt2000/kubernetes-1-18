# OpenShift FAQ for Container Ingress Controller

This page is under construction

**Problem:** CIS takes the node ip information from kube api in cluster mode. It should actually take these details from flannel as CIS creates the fdb entries in BIG-IP using these details. This issue is only seen when the nodes are multiple interfaces. 

**Solution:** Use Flannel annotations to get the node ip addresses. Add a new annotation for mac address and public IP. This needs to get added to each node. Please find the required annotations below. MAC address of the vxlan interface of that particular node.

```
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"<MAC>"}'
    flannel.alpha.coreos.com/public-ip: “<IP>”
```
In OpenShift edit the nodes yaml

```
oc get nodes -o yaml
and edit the yaml to add the annotations and apply
```
Github issue https://github.com/F5Networks/k8s-bigip-ctlr/issues/797

---
