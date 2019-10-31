# Install kubernetes 2.0 dashboard
When setting up dashboard you will need three components and a service account as displayed by the four files below

Get these files from https://github.com/justmeandopensource/kubernetes/tree/master/dashboard

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
  type: NodePort -- **changed from cluster port**
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 32323  -- **added listener port**
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
[kube@k8s-1-13-master root]$ kubectl describe sa dashboard-admin -n kube-system
Name:                dashboard-admin
Namespace:           kube-system
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   dashboard-admin-token-h5pvx
Tokens:              dashboard-admin-token-h5pvx
Events:              <none>

kubectl describe secret dashboard-admin-token-h5pvx -n kube-system
Name:         dashboard-admin-token-h5pvx
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 24576691-186d-11e9-9f7d-005056bb599e

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4taDVwdngiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMjQ1NzY2OTEtMTg2ZC0xMWU5LTlmN2QtMDA1MDU2YmI1OTllIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.QuxTQo7-Aa6TArBaWfbPJpTJ1RWJemHxcyqU6HJd5Twmrmfsg4BXwb31mymaGrfHgRR6mpfaGcQqFW1tLELu2dXLy-Q6d9hNouURo19jG1ZdGqYTGde-OwVgRgZqF3EqAOzM90A9nGcSCwTihlzzeYgseuLGwknYtUp5-70eVqy9dYvlEFCEBvoKJ02JRep7qIZHP3KAL-mqm2c2wYj6mbFyyR6dOSTVm9fcfexzl4r_8OZu8mChmdQId0ZROaCE2Kk8P0gP_vbMUT6SnQvgMXIuntrhf0NyyR9IY9PeQ5ahXl8FyaAYmnnvRq0gNg1zhr-tdvOXgeoTSXPfTV4V_Q
```
