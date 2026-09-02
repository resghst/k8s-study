# 概念篇(先讀這裡)

這個資料夾用**中文**、**短篇**幫你「先抓住觀念」,再去 [`notes/`](../notes/) 動手做。
每一課的順序都建議是:

> **先讀概念卡(這裡,5 分鐘)→ 再做 hands-on(`notes/`,親手跑指令)**

概念卡只回答四件事:**這是什麼、解決什麼問題、最小的心智模型、去哪練**。
細節、指令、預期輸出都在對應的 `notes/` 裡。

## 先看大方向(總覽)

先花十分鐘建立整體地圖,後面每一課才知道自己站在哪。

1. [K8s 到底在做什麼](big-picture.md) — 為什麼存在、宣告式模型、reconcile 迴圈
2. [核心物件全圖](object-map.md) — 一張圖看懂 Pod / Deployment / Service / Ingress / Config 怎麼串

## 再逐課抓概念

| 概念卡 | 一句話 | 動手做 |
| --- | --- | --- |
| [01 架構](01-architecture.md) | 控制平面與節點怎麼分工 | [notes/01](../notes/01-architecture.md) |
| [02 kubectl](02-kubectl.md) | 你跟叢集對話的唯一窗口 | [notes/02](../notes/02-kubectl.md) |
| [03 Pod](03-pods.md) | 最小的執行單位 | [notes/03](../notes/03-pods.md) |
| [04 Deployment](04-deployments.md) | 幫 Pod 找個管家(自癒/擴縮/更新) | [notes/04](../notes/04-deployments.md) |
| [05 Service](05-services.md) | 給會換 IP 的 Pod 一個固定地址 | [notes/05](../notes/05-services.md) |
| [06 Ingress](06-ingress.md) | 一個大門,依網址分流 | [notes/06](../notes/06-ingress.md) |
| [07 ConfigMap & Secret](07-config.md) | 設定與密碼跟 image 拆開 | [notes/07](../notes/07-config.md) |

## 怎麼用這份筆記

- **完全新手** → 依序讀總覽 → 逐課「概念卡 → notes」。
- **想動手** → 直接看 `notes/`,卡住再回來看概念卡。
- **忘記某個東西是什麼** → 查概念卡(短)或 [notes/objects.md](../notes/objects.md)(物件參考)。

環境還沒好?先去 [notes/00-environment.md](../notes/00-environment.md) 架一個能亂搞的 k3s 練習叢集。
