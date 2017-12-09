# 配置 CA 和產生 TLS 憑證

---

本次實驗你將使用 CloudFlare's PKI 工具, [cfssl](https://github.com/cloudflare/cfssl), 來提供 [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure), 然後使用它去建立Certificate Authority(CA), 並產生 TLS 憑證給以下組件使用: etcd, kube-apiserver, kubelet, 和 kube-proxy

## Certificate Authority

在這個部份會提供 Certificate Authority 來產生額外的 TLS 憑證

新建 CA 設定檔:


```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

新建 CA 憑證簽發請求文件:

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF
```
產生 CA 憑證和 私鑰:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

結果:

```
ca-key.pem
ca.pem
```

## client 與 server 憑證

這個部份你將會建立 client 與 server 的憑證給每個 Kubernetes 的組件, 建立一個 client 憑證 給Kubernetes `admin` 使用者


### The Admin Client Certificate

建立 `admin` client 憑證簽發請求文件:


```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

產生 `admin` client 憑證 和 私鑰:


```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```


結果:

```
admin-key.pem
admin.pem
```


### The Kubelet Client Certificates

Kubernetes 使用[special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/), 被稱作Node Authorizer, 這個是用來授權 來自[Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet)
的 API  請求。為了要通過 Node Authorizer  的授權, Kubelet 必須使用一個憑證屬名為`system:node:<nodeName>`, 來證明它屬於 `system:nodes` 的群集。

在這個部份將產生一個憑證給每個 Kubernetes 工作節點以符合Node Authorizer的需求。

建立 憑證以及私鑰 給每個 Kubernetes 工作節點:


```
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

結果: 

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem

```

### The kube-proxy Client Certificate



```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

建立 `kube-proxy` client 憑證和私鑰:



```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

結果:

```
kube-proxy-key.pem
kube-proxy.pem
```


### The Kubernetes API Server Certificate

`kubernetes-the-hard-way`的固定 IP 地址 會被含在 Kubernetes API Server 憑證裡

這將確保此憑證對遠端客戶端仍然有效

設置 `kubernetes-the-hard-way`的 固定 IP 地址:



```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

建立 Kubernetes API Server 憑證簽發請求文件:



```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

建立 Kubernetes API Server 憑證與私鑰:


```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

結果:

```
kubernetes-key.pem
kubernetes.pem
```

## Distribute the Client and Server Certificates

複製憑證以及私鑰到每個工作節點上:

```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```

複製憑證以及私鑰到每個控制節點上:


```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem ${instance}:~/
done
```

> `kube-proxy` 和 `kubelet` client 憑證將會被用來產生client 的授權設定檔, 我們將在下一個實驗中說明

Next: [配置和生成 Kubernetes 配置文件](05-kubernetes-configuration-files.md)



