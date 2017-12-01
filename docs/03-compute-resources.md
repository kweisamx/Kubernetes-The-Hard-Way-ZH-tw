# 準備計算資源
Kubernetes 需要一些機器去搭建管理 Kubernetes 的控制平台, 也需要一些工作節點(work node)讓 container 運行, 在這個實驗你將會準備計算資源, 透過single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones)來運行安全且高可用的 Kubernetes 叢集 

> 請確定 default compute zone 和 region 已照著 [事前準備](01-prerequisites.md#set-a-default-compute-region-and-zone)的設定步驟完成


## Networking

Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) 假設使用flat 
網路能讓每個 container 與節點都互相溝通。 在這邊我們不去提及 network policies ,一個用來控管 container 之間相互的連線, 或是連到外網的終端的機制


> 設定network policies 不在這次教學範圍內


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

### Kubernetes Public IP Address

分配固定的ＩＰ 地址, 被用來連接外部的負載平衡器至Kubernetes API Servers:


```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

驗證 `kubernetes-the-hard-way` 固定IP地址被 你的default compute region 建立出來:

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> 輸出為

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

## 計算節點



計算節點 將會使用[Ubuntu Server](https://www.ubuntu.com/server) 16.04, 原因是它對[cri-containerd container runtime](https://github.com/kubernetes-incubator/cri-containerd)有很好的支持 每個計算instance 會被分到一個私有 IP address 用以簡化Kubernetes 的建置



### Kubernetes Controllers

建立三個計算節點用以配置Kubernetes的控制平台

```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```


### Kubernetes Workers
每個worker 節點 需要一個Pod子網分配從Kubernetes 叢集CIDR的範圍, Pod的網路分配會在之後的容器網路章節做練習, 在運行時, `pod-cidr` 節點 會被用來暴露pod的網路給計算節點

> Kubernetes 叢集CIDR的範圍被定義在Controller Manager `--cluster-cidr` 參數, 在本次教學中我們會設定`10.200.0.0/16`, 將支援到254個子網

產生三個計算節點 用以配置 Kubernetes Worker節點

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### 驗證

列出所有在你的Default compute zone的計算節點


```
gcloud compute instances list
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

Next: [提供CA 和 產生 TLS 憑證](04-certificate-authority.md)
