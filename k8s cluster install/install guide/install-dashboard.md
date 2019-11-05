# Install kubernetes 2.0 dashboard
When setting up dashboard you will need multiple components and a service account

- kubernetes-dashboard.yaml
- sa_cluster_admin.yaml

```
##### Modify kubernetes-dashboard yaml 
Create kubernetes-dashboard. Change type to NodePort and add listener port
```
# ------------------- Dashboard Service ------------------- #
```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort -- changed from cluster port
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 32323  -- added listener port
  selector:
    k8s-app: kubernetes-dashboard   
```

```
kubectl create -f kubernetes-dashboard.yaml 
```
```
##### Check cluster information 
[kube@k8s-1-16-master install guide]$ kubectl get services --all-namespaces
NAMESPACE              NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
default                kubernetes                  ClusterIP   10.96.0.1        <none>        443/TCP                  19h
kube-system            kube-dns                    ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   19h
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.104.232.16    <none>        8000/TCP                 4m34s
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.105.221.248   <none>        443:32323/TCP            4m34s
```

Connect to the dashboard https://192.168.200.86:32323

##### Create service-account
To access the dashboard you need to create a service account
```
kubectl create -f sa_cluster_admin.yaml
```
###### Get secrets

```
kubectl describe sa dashboard-admin -n kube-system
Name:                dashboard-admin
Namespace:           kube-system
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   dashboard-admin-token-h5pvx
Tokens:              dashboard-admin-token-h5pvx
Events:              <none>
```
###### Get token to access the dashboard
```
[kube@k8s-1-16-master install guide]$ kubectl describe sa dashboard-admin -n kube-system
Name:                dashboard-admin
Namespace:           kube-system
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   dashboard-admin-token-t8vcl
Tokens:              dashboard-admin-token-t8vcl
Events:              <none>
```
```
[kube@k8s-1-16-master install guide]$ kubectl describe secret dashboard-admin-token-t8vcl -n kube-system
Name:         dashboard-admin-token-t8vcl
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: d4244343-941f-4312-9de3-3b145322eb16

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkFza3RpMUJCRTQ0ZTNqY3pWQmpOZHowZEtlOUdrOUJOcV9IdXdWaHc5NG8ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tdDh2Y2wiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZDQyNDQzNDMtOTQxZi00MzEyLTlkZTMtM2IxNDUzMjJlYjE2Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.bBu2DEMD7wbW8diQSzKff_eMDUYCM8RMGaVftfyIZ0s23U7AakJDOlKyrR1-4oDCI4pqrNwo-d_B_p_tUbr8XOqB2POMG-haWDx1vM9smptkHE-NL9YFi6_kubqXaDLwHYu72k-a3XfVFCKmtvyFcq13EO9mbLncE0-Agxl5or1Cvza0Ryp3qHHIRl8Q2SB8G6prnvw2KFcC5US0Eu7C2m5uO8WWfJtKdabA8-5v5HkmbqDUSuJCB9AUVy6rqwHqjv3q_ZOIWeVzVlytwkZ0lym7HIAHWjnY_z5AsmM5SqlvBY2u69E9W5LojR4pLqxvMBfPH_XCempdSnGGZOpClw
[kube@k8s-1-16-master install guide]$
```
