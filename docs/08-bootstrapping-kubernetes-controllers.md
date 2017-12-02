# 啟動 Kubernetes 控制平台

在本次實驗中你將會啟動整個Kubernetes平台, 透過對三個計算節點的配置以及建立高可用群集。

你也會建立外部負載均衡器去對外暴露Kubernetes API Servers 給遠端clients 。以下組件將會被安裝在每個節點上: Kubernetes API Server, Scheduler, 和 Controller Manager

## 事前準備

這次的指令必須在每個控制節點上使用:`controller-0`, `controller-1`, 與 `controller-2`。使用 `gcloud` 的指令登入每個控制節點。

例如:

```
gcloud compute ssh controller-0
```
## 建立 Kubernetes 控制平台

### 下載且安裝 Kubernetes Controller 執行檔




```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl"
```

安裝 Kubernetes 執行檔:


```
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
```

```
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```


### 設定 Kubernetes API Server


```
sudo mkdir -p /var/lib/kubernetes/
```

```
sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem encryption-config.yaml /var/lib/kubernetes/
```

節點內部的IP address 將被用來廣播 API server 給每個在群集裡的成員。 取得目前計算節點的內部IP address:

```
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

建立`kube-apiserver.service` systemd unit file:

```
cat > kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --admission-control=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --insecure-bind-address=127.0.0.1 \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/ca-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-ca-file=/var/lib/kubernetes/ca.pem \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 設定 Kubernetes Controller Manager

建立 `kube-controller-manager.service` systemd unit file:



```
cat > kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --leader-elect=true \\
  --master=http://127.0.0.1:8080 \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/ca-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
### 設定Kubernetes Scheduler

建立`kube-scheduler.service` systemd unit file:


```
cat > kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --leader-elect=true \\
  --master=http://127.0.0.1:8080 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 啟動 Controller 服務

```
sudo mv kube-apiserver.service kube-scheduler.service kube-controller-manager.service /etc/systemd/system/
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
```

```
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

> 請等待 10秒 的Kubernetes API Server 初始化時間

### 驗證

```
kubectl get componentstatuses
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

> 記得上述的指令都要執行每個控制節點上: `controller-0`, `controller-1`, and `controller-2`

## RBAC - Kubelet 授權

在這個部份你將會設定 RBAC 許可, 用來允許Kubernetes API Server 得以請求每個worker節點的Kubelet API

請求 Kubeket API 用以獲取相關的資源, 例如: metrics, logs, 和在每個Pod裡執行指令

> 這份教學設置 Kubeket `--authorization-mode` flag 給 `Webhook`。Webhook模式使用[SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) api 去決定授權

```
gcloud compute ssh controller-0
```

建立 `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) 用允許 請求Kubelet API 以及執行許多任務用來管理Pods:


```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```
Kubernetes API Server 授權 Kubelet 為`kubernetes` user, 使用 client 憑證, 此憑證用`--kubelet-client-certificate` flag 來定義

連接`system:kube-apiserver-to-kubelet` ClusterRole 到`kubernetes` user:

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## Kubernetes 前端負載均衡器

在這個部份你將會建立外部負載均衡器, 並建立在 Kubernetes API Servers 的前端。 `kubernetes-the-hard-way` 固定IP address 將會被配置在這負載均衡器上

> 在這份教學中建立的計算節點並沒有權限, 以至於無法完成這個部份。執行以下的指令去新建計算節點

建立外部負載均衡器的網路資源:



```
gcloud compute target-pools create kubernetes-target-pool
```

```
gcloud compute target-pools add-instances kubernetes-target-pool \
  --instances controller-0,controller-1,controller-2
```

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(name)')
```

```
gcloud compute forwarding-rules create kubernetes-forwarding-rule \
  --address ${KUBERNETES_PUBLIC_ADDRESS} \
  --ports 6443 \
  --region $(gcloud config get-value compute/region) \
  --target-pool kubernetes-target-pool
```

### 驗證

取得 `kubernetes-the-hard-way` 固定IP address:

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```


傳送一個 Kubernetes版本資訊的HTTP 請求:

```
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> 輸出為


```
{
  "major": "1",
  "minor": "8",
  "gitVersion": "v1.8.0",
  "gitCommit": "6e937839ac04a38cac63e6a7a306c5d035fe7b0a",
  "gitTreeState": "clean",
  "buildDate": "2017-09-28T22:46:41Z",
  "goVersion": "go1.8.3",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Next: [啟動 Kubernetes Worker 節點](09-bootstrapping-kubernetes-workers.md)
