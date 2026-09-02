# 04 Deployment(概念)

> 一句話:**Deployment 是 Pod 的管家——幫它自癒、擴縮、還能不斷線換版本。**

## 解決什麼問題

上一課的痛點:裸 Pod 被刪就回不來。你需要有人「一直盯著,少了就補」。Deployment 就是那個角色。

## 心智模型

三層,各司其職:

```
Deployment  管版本(換 image 就開新版本)
   └─ ReplicaSet  管數量(維持 N 份)
        └─ Pod  真正在跑的東西
```

- 你宣告「我要 3 份」,**ReplicaSet 數數量**,少一個就補一個 → **自癒**。
- 改數字 = **擴縮**(同一個 ReplicaSet)。
- 改 image = **滾動更新**:開一個新 ReplicaSet,一個一個換過去,舊的降到 0 但**留著當退路** → 可以一鍵 `rollout undo` 回滾。

## 一件事看穿全局

**「擴縮」和「更新」的差別**:改數量不會換 ReplicaSet;改模板(image 等)會用 `pod-template-hash` 認出「這是新版本」,開新的 ReplicaSet。看到兩個 ReplicaSet 此消彼長,就是在滾動更新。

## 一句話記住

**幾乎所有 app 都用 Deployment 跑,不裸開 Pod。** 它給你自癒 + 擴縮 + 安全換版本。

---
🔧 動手做:[notes/04-deployments.md](../notes/04-deployments.md)
← [03 Pod](03-pods.md)｜→ [05 Service](05-services.md)
