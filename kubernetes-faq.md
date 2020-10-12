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

## Manage node labels

**Problem:** When using nodeport by default all nodes from the cluster will be added to the pool. In most cases you only want to add the worker nodes and exclude master nodes. To exclude master nodes using the label node-role.kubernetes.io/node parameter

**Solution:** Use the label node to create a new label for the node. This works in conjunction with the node-label-selector configured in CIS to only add nodes to the pool with the associated . In this quick start guide CIS will only add the nodes with label worker to the pool. Excluding the master node from the pool

```
kubectl label nodes k8s-1-18-node1.example.com f5role=worker
kubectl label nodes k8s-1-18-node2.example.com f5role=worker
```

Show the node labels

```
kubectl get nodes
NAME                         STATUS   ROLES    AGE     VERSION
k8s-1-18-master.example.com   Ready    master   7d19h   v1.18.0
k8s-1-18-node1.example.com    Ready    f5role   7d19h   v1.18.0
k8s-1-18-node2.example.com    Ready    f5role   7d19h   v1.18.0
args:
- "--bigip-username=$(BIGIP_USERNAME)"
- "--bigip-password=$(BIGIP_PASSWORD)"
- "--bigip-url=192.168.200.92"
- "--bigip-partition=k8s"
- "--namespace=default"
- "--pool-member-type=nodeport"
- "--log-level=DEBUG"
- "--insecure=true"
- "--manage-ingress=false"
- "--manage-routes=false"
- "--manage-configmaps=true"
- "--agent=as3"
- "--as3-validation=true"
- "--node-label-selector=f5role=worker" - set the node label
```

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