# 配置 Pod 網路路由

Pods 排程到節點上會從節點的Pod CIDR 範圍裡挑出一個IP address。由於missing network [routes](https://cloud.google.com/compute/docs/vpc/routes)導致這些pods並不能與屬於其他節點的pods溝通。

這次實驗我們將會建立一個路由給每個worker 節點, 幫助對應節點上 Pod CDIR 範圍 到此節點的 內部IP address。

> [其他方法](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)來實做Kubernetes 網路模型

## 路由表

在這個部份你將會收集必要的資訊用以建立`kubernetes-the-hard-way` VPC 網路的路由

列出每個worker 節點的內部IP address 和 Pod CIDR 範圍:



```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute instances describe ${instance} \
    --format 'value[separator=" "](networkInterfaces[0].networkIP,metadata.items[0].value)'
done
```
> 輸出為

```
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## 路由

為每個worker節點建立網路路由:

```
for i in 0 1 2; do
  gcloud compute routes create kubernetes-route-10-200-${i}-0-24 \
    --network kubernetes-the-hard-way \
    --next-hop-address 10.240.0.2${i} \
    --destination-range 10.200.${i}.0/24
done
```

列出`kubernetes-the-hard-way` VPC 網路的路由表:

```
gcloud compute routes list --filter "network: kubernetes-the-hard-way"
```

> 輸出為


```
NAME                            NETWORK                  DEST_RANGE     NEXT_HOP                  PRIORITY
default-route-77bcc6bee33b5535  kubernetes-the-hard-way  10.240.0.0/24                            1000
default-route-b11fc914b626974d  kubernetes-the-hard-way  0.0.0.0/0      default-internet-gateway  1000
kubernetes-route-10-200-0-0-24  kubernetes-the-hard-way  10.200.0.0/24  10.240.0.20               1000
kubernetes-route-10-200-1-0-24  kubernetes-the-hard-way  10.200.1.0/24  10.240.0.21               1000
kubernetes-route-10-200-2-0-24  kubernetes-the-hard-way  10.200.2.0/24  10.240.0.22               1000
```



Next: [部屬 DNS 擴展](12-dns-addon.md)
