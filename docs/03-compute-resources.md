# 準備計算資源
Kubernetes 需要一些機器去搭建管理 Kubernetes 的控制平台, 也需要一些工作節點(work node)讓 container 運行, 在這個實驗你將會準備計算資源, 透過single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones)來運行安全且高可用的 Kubernetes 叢集 

> 請確定 default compute zone 和 region 已照著 [事前準備](01-prerequisites.md#set-a-default-compute-region-and-zone)的設定步驟完成


## Networking

Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) 假設使用flat 
網路能讓每個 container 與節點都互相溝通。 在這邊我們不去提及 network policies ,一個可以控管 container 之間相互的連線, 或是連到外網的終端


> 設定network policies 已超出這次教學範圍


### Virtual Private Cloud Network

這個部份會搭建一個可靠的 [Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC) network 來搭建我們 Kubernetes 叢集

產生一個自訂 kubernetes-the-hard-way 的 網路環境


```
gcloud compute networks create kubernetes-the-hard-way --mode custom
```

一個子網必須提供足夠的虛擬 IP , 用以分配給 Kubernetes 叢集的每個節點

在`kubernetes-the-hard-way` VPC network產生`kubernetes`子網,


```
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
```

> `10.240.0.0/24` IP address範圍, 可以分配 254 計算節點

### Firewall Rules


建立一個防火牆規則允許內部網路可以通過所有的網路協定:

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```
建立一個防火牆規則允許外部SSH, ICMP, HTTPS等連線

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```

>  [外部負載均衡伺服器](https://cloud.google.com/compute/docs/load-balancing/network/) 被用來暴露 Kubernetes API Servers 給遠端Client

列出所有防火牆在`kubernetes-the-hard-way` VPC network的規則：

```
gcloud compute firewall-rules list --filter "network: kubernetes-the-hard-way"
```

> 輸出為

```
NAME                                         NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
kubernetes-the-hard-way-allow-external       kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
kubernetes-the-hard-way-allow-internal       kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
```

