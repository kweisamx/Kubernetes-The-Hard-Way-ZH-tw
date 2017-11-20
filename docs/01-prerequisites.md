# 事前準備

## Google Cloud Platform

這份教學文件使用了[Google Cloud Platform](https://cloud.google.com/)當作kubernetes叢集的環境平台。[註冊](https://cloud.google.com/free/) 即可獲得300美元的試用金。
[估計花費](https://cloud.google.com/products/calculator/#id=78df6ced-9c50-48f8-a670-bc5003f2ddaa)以完成教學: 每小時$0.22 (每天$5.39 ).

> 教學的計算資源會超出試用金的額度。

## Google Cloud Platform SDK

### 安裝 Google Cloud SDK
按照Google Cloud SDK [documentation](https://cloud.google.com/sdk/)的步驟去安裝以及設定`gcloud`的指令
驗證Google Cloud SDK 版本為 173.0.0 或更高:


```
gcloud version
```

### 設定一個 Default Compute Region 和 Zone

這個部份假設你的 defalut compute region 與 zone都已經被設定好

如果你第一次使用`gcloud`指令工具,`init` 是一個最簡單的方式:

```
gcloud init
```

手動設定default compute region:

```
gcloud config set compute/region us-west1
```
設定compute zone
```
gcloud config set compute/zone us-west1-c
```

> 使用 `gcloud compute zones list` 指令來查詢 其他的region 和 zone

Next: [安裝 Client 工具](02-client-tools.md)



