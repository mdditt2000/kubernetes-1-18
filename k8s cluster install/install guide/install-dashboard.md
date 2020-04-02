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
[kube@k8s-1-18-master install guide]$ kubectl get services --all-namespaces
NAMESPACE              NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default                kubernetes                  ClusterIP   10.96.0.1       <none>        443/TCP                  103m
kube-system            kube-dns                    ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   103m
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.105.213.41   <none>        8000/TCP                 16s
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.103.114.93   <none>        443:32323/TCP            16s
```

Connect to the dashboard https://192.168.200.93:32323

##### Create service-account
To access the dashboard you need to create a service account
```
kubectl create -f sa_cluster_admin.yaml
```
###### Get secrets

```
[kube@k8s-1-18-master install guide]$ kubectl describe sa dashboard-admin -n kube-system
Name:                dashboard-admin
Namespace:           kube-system
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   dashboard-admin-token-b7dwm
Tokens:              dashboard-admin-token-b7dwm
Events:              <none>
```
###### Get token to access the dashboard
```
[kube@k8s-1-18-master install guide]$ kubectl describe secret dashboard-admin-token-b7dwm -n kube-system
Name:         dashboard-admin-token-b7dwm
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: d2630b3e-23c0-4ad6-9ab0-299d1316d001

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjZ0OXVUTEJPVjE0MGFGVXhaai14OVlpbGRmd0g4bmRBS0tNVWotd0VCeFEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tYjdkd20iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZDI2MzBiM2UtMjNjMC00YWQ2LTlhYjAtMjk5ZDEzMTZkMDAxIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.rvSCQyyXP6cEI9MtsrtbR9n-fg4aZYUIrvEt0XyRiuJ9rX__z6AW33edLwKammnuNZMiBk7dao9GbV1xNhxSxghtUzJxRSFZ_mN_XafGUjgqNxfHcGMOEKGmLpgly749kZMvhOFUfyZO4KPJdq4PTEGqsHEDKh91JiJBjHLlypBEEQgqBKpoRbq5SLPpASyrNqKZmCeQbNPsQQ4mo-4aWT1RLvFfydU-I_1maIfSJhIk3apZQohQS6jpJYMmD_4RuHyzLxnacydG5InDSSydCrZgNvA19aFcTjHVvEZbX1mZt8fgCu7xumBlKQYX3Ihi6G5oWGTOqvSsaXlfkr6QSw
[kube@k8s-1-18-master install guide]$ ^C

```
