# 07 ConfigMap & Secret(概念)

> 一句話:**把設定和密碼從 image 裡拆出來,Pod 啟動時才餵進去。**

## 解決什麼問題

前面設定都寫死在 image 裡。但真實 app 有一堆設定(網址、開關)和密碼(password、API key),烤進 image 有兩個問題:改一個值就要重 build、而且密碼會外洩。要把「設定」跟「程式」分開。

## 心智模型

- **ConfigMap** 裝**非機密**設定;**Secret** 裝**機密**資料。
- Pod **用名字引用**它們(不是把值抄進去),一份設定可餵很多 Pod。
- 兩種注入方式,行為不同:

| 注入方式 | 改了設定後 |
| --- | --- |
| 當**環境變數** | **凍結**,要重啟 Pod 才生效 |
| 掛成**檔案(volume)** | **自動更新**(適合熱重載) |

## 一件事看穿全局

**Secret 不是加密!** 它的值只是 **base64 編碼**,`base64 -d` 隨手還原。它跟 ConfigMap 的差別在「意圖 + 權限控管(RBAC、不進 log、可選 etcd 加密)」,但本身不幫你加密。**所以真正的密碼別明碼 commit 進 git。**

## 一句話記住

**設定 → ConfigMap,密碼 → Secret,兩者都跟 image 解耦、用名字引用。base64 ≠ 加密。**

---
🔧 動手做:[notes/07-config.md](../notes/07-config.md)
← [06 Ingress](06-ingress.md)｜回到 [概念索引](README.md)
