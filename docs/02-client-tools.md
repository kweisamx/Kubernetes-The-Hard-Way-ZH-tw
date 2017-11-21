# 安裝Client 工具

---

本次實驗你將會安裝一些實用的指令工具, 用來完成這次的整份教學 :  [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl)

從[cfssl repository](https://pkg.cfssl.org)下載`cfssl` and `cfssljson` 與安裝

### OS X

```
curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_darwin-amd64
curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_darwin-amd64
```

```
chmod +x cfssl cfssljson
```

```
sudo mv cfssl cfssljson /usr/local/bin/
```

### Linux

```
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
```

```
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
```

```
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
```

```
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

### 驗證

驗證`cfssl`的版本為1.2.0或是更高

```
cfssl version
```

> output

```
Version: 1.2.0
Revision: dev
Runtime: go1.6
```
> cfssljson的指令工具沒有提供方法來列出版本

## 安裝kubectl

`kubectk`指令工具是用來與 Kubernetes API Server溝通的, 下載並安裝`kubectl` ,可在官方取得
### OS X

```
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/darwin/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Linux

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### 驗證

驗證`kubectl`的安裝版本為 1.8.0 或是更高
```
kubectl version --client
```

> 輸出為

```
Client Version: version.Info{Major:"1", Minor:"8", GitVersion:"v1.8.0", GitCommit:"6e937839ac04a38cac63e6a7a306c5d035fe7b0a", GitTreeState:"clean", BuildDate:"2017-09-28T22:57:57Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"darwin/amd64"}
```

Next: [準備計算資源](03-compute-resources.md)
