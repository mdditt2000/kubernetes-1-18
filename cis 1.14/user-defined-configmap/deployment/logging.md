# Container Ingress Services 1.9.1 logging
Loggging is key for debug. I have create this page to make logging container ingress simple 

## Locate BigIP Controller Pod name
Determine the pod name of the BigIP controller for logging
```
[kube@k8s-1-13-master cis-1-9]$ kubectl get pod -n kube-system
NAME                                                       READY   STATUS    RESTARTS   AGE
k8s-bigip-ctlr-deployment-55b8fc9fb5-m4r2h                 1/1     Running   0          10m
```
## View logging
```
kubectl log -f k8s-bigip-ctlr-deployment-55b8fc9fb5-m4r2h -n kube-system
```
# View logging last 20 lines etc
```
kubectl logs --tail=20
```
# Show all logs from pod nginx written in the last hour
kubectl logs --since=1h