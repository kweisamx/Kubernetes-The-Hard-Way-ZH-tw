# 刪除集群

在本次實驗終將移除在本教學中全部的計算資源

## 計算節點
刪除所有的控制節點 和 worker 節點:


```
gcloud -q compute instances delete \
  controller-0 controller-1 controller-2 \
  worker-0 worker-1 worker-2
```

## 網路

刪除外部負載均衡器以及網路資源:

```
gcloud -q compute forwarding-rules delete kubernetes-forwarding-rule \
  --region $(gcloud config get-value compute/region)
```

```
gcloud -q compute target-pools delete kubernetes-target-pool
```

Delete the `kubernetes-the-hard-way` static IP address:

```
gcloud -q compute addresses delete kubernetes-the-hard-way
```

刪除`kubernetes-the-hard-way` 防火牆規則:

```
gcloud -q compute firewall-rules delete \
  kubernetes-the-hard-way-allow-nginx-service \
  kubernetes-the-hard-way-allow-internal \
  kubernetes-the-hard-way-allow-external
```

刪除Pod 網路路由:

```
gcloud -q compute routes delete \
  kubernetes-route-10-200-0-0-24 \
  kubernetes-route-10-200-1-0-24 \
  kubernetes-route-10-200-2-0-24
```

刪除 `kubernetes` 子網:

```
gcloud -q compute networks subnets delete kubernetes
```

刪除 `kubernetes-the-hard-way` network VPC:

```
gcloud -q compute networks delete kubernetes-the-hard-way
```
