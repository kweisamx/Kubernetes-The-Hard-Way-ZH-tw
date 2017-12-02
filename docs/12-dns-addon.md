# 部屬DNS群集插件

在本次實驗中將會部屬[DNS 插件](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)用來提供DNS, 為集群內的應用程式提供服務發現。


## DNS 群集插件

部屬 `kube-dns` 群集插件:

```
kubectl create -f https://storage.googleapis.com/kubernetes-the-hard-way/kube-dns.yaml
```

> 輸出為

```
serviceaccount "kube-dns" created
configmap "kube-dns" created
service "kube-dns" created
deployment "kube-dns" created
```

列出`kube-dns` deploment 的pods:

```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> 輸出為

```
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-3097350089-gq015   3/3       Running   0          20s
kube-dns-3097350089-q64qc   3/3       Running   0          20s
```

## 驗證

建立一個`busybox` deployment:

```
kubectl run busybox --image=busybox --command -- sleep 3600
```

列出`busybox` deployment 的 pod


```
kubectl get pods -l run=busybox
```

> 輸出為


```
NAME                       READY     STATUS    RESTARTS   AGE
busybox-2125412808-mt2vb   1/1       Running   0          15s
```

取得`busybox` pod 的全名:

```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

執行在`busybox` pod 的 DNS 查詢服務


```
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> 輸出為

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Next: [煙霧測試](13-smoke-test.md)
