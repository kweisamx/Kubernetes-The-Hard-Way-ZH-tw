#  啟動 Kubernetes Worker 節點
在本次實驗中,你將會新建並啟動三個Kubernetes work 節點。

以下組件將會被安裝到各個節點: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [cri-containerd](https://github.com/kubernetes-incubator/cri-containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies)


## 事前準備

這次的指令必須在每個 worker 節點上使用: `worker-0`, `worker-1`, and `worker-2`。使用 `gcloud` 的指令登入每個 worker 節點。

例如:

```
gcloud compute ssh worker-0
```

## 建立 Kubernetes Worker 節點

安裝 OS 的相關套件:


```
sudo apt-get -y install socat
```

> socat 執行檔可支援 `kubectl port-forward` 指令

### 下載並安裝 worker 執行檔


```
wget -q --show-progress --https-only --timestamping \
  https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
  https://github.com/kubernetes-incubator/cri-containerd/releases/download/v1.0.0-alpha.0/cri-containerd-1.0.0-alpha.0.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubelet
```

建立 安裝目錄:


```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

安裝 worker 執行檔:

```
sudo tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
```

```
sudo tar -xvf cri-containerd-1.0.0-alpha.0.tar.gz -C /
```

```
chmod +x kubectl kube-proxy kubelet
```

```
sudo mv kubectl kube-proxy kubelet /usr/local/bin/
```


### 設定 CNI 網路

取得目前Pod CDIR 範圍給當前的計算節點:

```
POD_CIDR=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr)
```

建立 `bridge` network 設定檔:


```
cat > 10-bridge.conf <<EOF
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

建立 `loopback` network 設定檔:



```
cat > 99-loopback.conf <<EOF
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF
```
移動網路相關部份的設定擋到CNI的設定資料夾目錄下:
```
sudo mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
```

### 設定 Kubelet



```
sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
```

```
sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
```

```
sudo mv ca.pem /var/lib/kubernetes/
```

建立 `kubelet.service` systemd unit file:

```
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=cri-containerd.service
Requires=cri-containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --allow-privileged=true \\
  --anonymous-auth=false \\
  --authorization-mode=Webhook \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --cluster-dns=10.32.0.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/cri-containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --pod-cidr=${POD_CIDR} \\
  --register-node=true \\
  --require-kubeconfig \\
  --runtime-request-timeout=15m \\
  --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.pem \\
  --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 設定 Kube-Proxy 

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

建立 `kube-proxy.service` systemd unit file:

```
cat > kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --cluster-cidr=10.200.0.0/16 \\
  --kubeconfig=/var/lib/kube-proxy/kubeconfig \\
  --proxy-mode=iptables \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```



### 啟動 worker 服務


```
sudo mv kubelet.service kube-proxy.service /etc/systemd/system/
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable containerd cri-containerd kubelet kube-proxy
```

```
sudo systemctl start containerd cri-containerd kubelet kube-proxy
```
> 記得上述的指令都要執行每個 worker 節點上: `worker-0`, `worker-1`, 和 `worker-2`


## 驗證

登入 其中一個控制節點:

```
gcloud compute ssh controller-0
```

列出目前以註冊的Kubernetes 節點:


```
kubectl get nodes
```

> 輸出為


```
NAME       STATUS    ROLES     AGE       VERSION
worker-0   Ready     <none>    1m        v1.8.0
worker-1   Ready     <none>    1m        v1.8.0
worker-2   Ready     <none>    1m        v1.8.0
```


Next: [配置 Kubectl](10-configuring-kubectl.md)
