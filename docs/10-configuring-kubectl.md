# 配置 Kubectl

在本次實驗中你將會建立基於`admin` user 憑證的kubeconfig檔給`kubectl`指令使用

> 在這個實驗同個目錄中, 運行指令來產生admin client憑證

## Admin Kubernetes 設定檔

每一個kubeconfig 需要一個Kuberntes API Server 連接, 為了支援高可用, IP address被分配到外部負載均衡器, Kubernetes API Server 將部署在負載均衡器之後

設定kubernetes-the-hard-way 的固定IP address:

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

為 `admin` user 建立認證用kubeconfig檔:


```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443
```

```
kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem
```

```
kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin
```

```
kubectl config use-context kubernetes-the-hard-way
```


## 驗證

檢查遠端Kubernetes 群集的健康狀況:

```
kubectl get componentstatuses
```

> 輸出為

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

列出遠端kubernetes cluster的節點:


```
kubectl get nodes
```

> 輸出為


```
NAME       STATUS    ROLES     AGE       VERSION
worker-0   Ready     <none>    2m        v1.8.0
worker-1   Ready     <none>    2m        v1.8.0
worker-2   Ready     <none>    2m        v1.8.0
```


Next: [配置 Pod 網路路由](11-pod-network-routes.md)


