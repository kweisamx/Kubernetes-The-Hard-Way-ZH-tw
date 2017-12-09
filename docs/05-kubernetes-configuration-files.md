# 配置和生成 Kubernetes 配置文件
在此次實驗中, 你會建立[Kubernetes 設定檔](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), 又被稱作 kubeconfigs, 用來使Kubernetes client能搜尋並認證Kubernetes API Server


## Client 認證設定
這個步驟你將會建立kubeconfig給`kubelet` 和`kube-proxy`

> `scheduler` 和 `controller manager ` 進入Kubernetes API Servers 透過一個不安全的API port , 這個port 並不需要認證, 所以這個port只允許來自本地端的請求進入

### Kubernetes 公有IP address
每一個kubeconfig 需要一個Kuberntes API Server 連接, 為了支援高可用, IP address被分配到外部負載均衡器, Kubernetes API Server 將部署在負載均衡器之後

設定`kubernetes-the-hard-way` 的固定IP address

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

### kubelet Kubernetes 設定檔

當建立kubeconfig給kubelet, client的憑證對應到kubelet 的 node 一定會被使用

這是為了確保kubelet 確實的被Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/)授權

建立kubeconfig給每個work node:

```
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```
結果：

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### kube-proxy Kubernetes 設定檔

建立kubeconfig 給 `kube-proxy`:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig
```

```
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
```

```
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
```

```
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

### 分配kubernetes設定檔

複製 `kubelet` 與 `kube-proxy` kubeconfig設定檔 到每個work node上：

```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```



Next: [配置和生成密鑰](06-data-encryption-keys.md)
