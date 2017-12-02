# Kubernetes The Hard Way

---
此文件翻譯來源:[kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

如有翻譯不好的地方或文字上的錯誤, 都歡迎提出 issue 或是 PR

---

這份教學將帶領你走上安裝kubernetes的艱辛之路。這份文件不適用於想要一鍵自動化部屬kubernetes叢集的人。如果你想要輕鬆部屬, 可以參考[Google Container Engine](https://cloud.google.com/container-engine) 或[Getting Started Guides](http://kubernetes.io/docs/getting-started-guides/)

Kubernetes The Hard Way 是個學習的最佳方式, 會花上許多時間確保你真正了解每項組件的任務以及需求,去搭建整個kubernetes 叢集
> 這份文件不應該出現在production 文件中, 也無獲得kubernetes社區的許多支持, 但這都不影響你想真正了解Kubernetes的決心！

---

## 誰適合看這份文件？
這份文件的目標是給那些計畫要使用kubernetes 當作production環境的人, 並想了解每個有關kubernetes的環節以及他們如何運作的

## 有關叢集的詳細資訊
Kubernetes The Hard Way 將引導你建立高可用的Kubernetes的叢集, 包括每個組件之間的加密以及RBAC認證

* [Kubernetes](https://github.com/kubernetes/kubernetes) 1.8.0
* [cri-containerd Container Runtime](https://github.com/kubernetes-incubator/cri-containerd) 1.0.0-alpha.0
* [CNI Container Networking](https://github.com/containernetworking/cni) 0.6.0
* [etcd](https://github.com/coreos/etcd) 3.2.8

## 實驗步驟

這份教學假設你已經有辦法登入[Google Cloud Platform](https://cloud.google.com), GCP被用來作為這篇教學的基礎需求,你也可以將這篇教學應用在其他平台上

* [事前準備](docs/01-prerequisites.md)
* [安裝 Client 工具](docs/02-client-tools.md)
* [準備計算資源](docs/03-compute-resources.md)
* [提供 CA 和產生 TLS 憑證](docs/04-certificate-authority.md)
* [建立認證用Kubernetes 設定檔](docs/05-kubernetes-configuration-files.md)
* [建立資料加密設定檔與密鑰](docs/06-data-encryption-keys.md)
* [啟動etcd 群集](docs/07-bootstrapping-etcd.md)
* [啟動 Kubernetes 控制平台](docs/08-bootstrapping-kubernetes-controllers.md)
* [啟動 Kubernetes Worker 節點](docs/09-bootstrapping-kubernetes-workers.md)
* [遠端請求Kubectl相關設定](docs/10-configuring-kubectl.md)
* [提供Pod網路路由](docs/11-pod-network-routes.md)
* [部屬DNS群集插件](docs/12-dns-addon.md)
* [煙霧測試](docs/13-smoke-test.md)
* [Cleaning Up](docs/14-cleanup.md)
