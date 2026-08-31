# 核心概念

## Pod

Pod 是 Kubernetes 最小的部署單位,裡面可以有一個或多個 container,共用網路與儲存。

## ReplicaSet

ReplicaSet 確保指定數量的 Pod 副本持續運行。通常不直接建立,而是由 Deployment 管理。

## Deployment

Deployment 提供宣告式更新,支援滾動更新與回滾。實務上大多透過 Deployment 來管理無狀態服務。

## Service

Service 為一組 Pod 提供穩定的存取入口。常見型別:

- `ClusterIP`:叢集內部存取(預設)
- `NodePort`:透過節點連接埠對外
- `LoadBalancer`:由雲端供應商配置外部負載平衡器
