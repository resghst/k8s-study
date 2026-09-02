# 05 Service(概念)

> 一句話:**Service 給一組「會換 IP 的 Pod」一個永遠不變的地址,並自動分流。**

## 解決什麼問題

上一課的痛點:Deployment 每次重建 Pod,IP 都會變。別人要連你,不可能一直追著變動的 IP 跑。Service 就是擋在前面那個固定的點。

## 心智模型

- Service 有一個**固定的虛擬 IP(ClusterIP)**和一個 **DNS 名字**,永遠不變。
- 它靠**標籤 selector**(例如 `app=web`)自動找到背後的 Pod,名單(Endpoints)由控制器**自動維護**——Pod 生死它自己更新,你不用填 IP。
- 打到 Service 的流量,由 `kube-proxy` 在核心層**逐連線分散**到後面的 Pod。

```
Client → (DNS: web) → Service 固定IP → 自動分流 → Pod / Pod / Pod
```

## 一件事看穿全局

**砍掉一個 Pod,新 Pod 換了新 IP,但 Service 的 IP 完全不動,Endpoints 自動換上新 IP。** 這就是「固定地址 + 自動追蹤」,上一課的痛點到此解決。

## 三種型別(由內到外)

| 型別 | 誰連得到 |
| --- | --- |
| `ClusterIP`(預設) | 只有叢集內部 |
| `NodePort` | 叢集外,用 `節點IP:高位埠` |
| `LoadBalancer` | 叢集外,用一個對外 IP(雲環境) |

## 一句話記住

**Service = 一組 Pod 的固定門牌 + 自動分流。** 它做的是 L4(IP/port),不看網址。

---
🔧 動手做:[notes/05-services.md](../notes/05-services.md)
← [04 Deployment](04-deployments.md)｜→ [06 Ingress](06-ingress.md)
