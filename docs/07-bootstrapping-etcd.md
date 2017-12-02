# 啟動 etcd 群集

Kubernetes 組件是屬於無狀態並且將群集狀態儲存到[etcd](https://github.com/coreos/etcd)

在這次實驗中你將會啟動三個節點的etcd群集, 並且做一些相關設定讓群集高可用與建立遠端安全連線

## 事前準備

這次的指令必須在每個控制節點上使用:`controller-0`, `controller-1`, 與 `controller-2`。使用 `gcloud` 的指令登入每個控制節點。

例如:

```
gcloud compute ssh controller-0
```

## 啟動一個etcd群集的成員

### 下載並安裝 etcd 的執行檔

從[coreos/etcd](https://github.com/coreos/etcd) GitHub專案下載etcd開放的執行檔:


```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.2.8/etcd-v3.2.8-linux-amd64.tar.gz"
```

解壓縮並安裝`etcd` server 與 `etcdctl`指令工具 :

```
tar -xvf etcd-v3.2.8-linux-amd64.tar.gz
```

```
sudo mv etcd-v3.2.8-linux-amd64/etcd* /usr/local/bin/
```
### 設定etcd Server

```
sudo mkdir -p /etc/etcd /var/lib/etcd
```

```
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

節點的內部IP address 將被用來接收 client 的請求並且聯繫整個etcd群集。 取得目前計算節點的內部IP address:


```
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

每個 etcd 成員必須有一個特別的名字在整個 etcd 的群集裡。 設定 etcd 名字去對應到目前計算節點的 hostname :

```
ETCD_NAME=$(hostname -s)
```

建立  `etcd.service`  的systemd unit file:



```
cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 啟動 etcd Server


```
sudo mv etcd.service /etc/systemd/system/
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable etcd
```

```
sudo systemctl start etcd
```

> 記得上述的指令都要執行每個控制節點上: `controller-0`, `controller-1`, and `controller-2`


## 驗證

列出etcd的群集成員:


```
ETCDCTL_API=3 etcdctl member list
```

> 輸出

```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379
```
Next: [啟動 Kubernetes 控制平台](08-bootstrapping-kubernetes-controllers.md)


