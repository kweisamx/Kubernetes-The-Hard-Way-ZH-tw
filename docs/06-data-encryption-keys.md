# 配置和生成密鑰

Kubernetes 儲存許多的資料, 像是群集狀態, 應用設定, 以及secrets。而Kubernetes 支援群集資料加密的相關功能。

在這次實驗你將會建立加密密鑰以及[加密設定檔](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) 來幫助加密Kubernetes Secests。

## 加密密鑰

建立加密密鑰:

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## 加密設定檔
建立 `encryption-config.yaml` 加密的設定檔:

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

複製 `encryption-config.yaml` 加密設定檔到每個控制節點:
```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp encryption-config.yaml ${instance}:~/
done
```


Next: [啟動etcd 群集](07-bootstrapping-etcd.md)
