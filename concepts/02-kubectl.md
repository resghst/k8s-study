# 02 kubectl(概念)

> 一句話:**kubectl 是你跟叢集對話的唯一窗口,它只是幫你把指令翻成 API 呼叫。**

## 解決什麼問題

叢集的目標狀態都存在控制平面裡,你需要一個工具去「看現在怎樣」和「說我要怎樣」。kubectl 就是那支遙控器。

## 心智模型

幾乎每個指令都是同一個文法:

```
kubectl <動詞> <資源類型> <名字> [選項]
kubectl   get     pods       web      -o yaml
```

- **看**:`get`(列表)、`describe`(細節 + 事件)、`logs`(日誌)
- **改**:`apply`(照檔案調成這樣,宣告式,最常用)、`delete`
- **查自己**:`explain`(某欄位是什麼)、`api-resources`(有哪些資源)、`--help`

## 一件事看穿全局

記住 `apply` 的精神:它比對的是「**最終長相**」不是「執行動作」。同一個檔案 `apply` 幾次結果都一樣,不會重複建——這正好對應 K8s 的宣告式本質。

## 一句話記住

**看用 `get`/`describe`/`logs`,改用 `apply`,不懂用 `explain`。** 這四個先熟。

---
🔧 動手做:[notes/02-kubectl.md](../notes/02-kubectl.md)
← [01 架構](01-architecture.md)｜→ [03 Pod](03-pods.md)
