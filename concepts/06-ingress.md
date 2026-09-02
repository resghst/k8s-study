# 06 Ingress(概念)

> 一句話:**Ingress 是整個叢集的統一大門,依「網址」把 HTTP 流量分給不同 Service。**

## 解決什麼問題

上一課的痛點:對外只有 NodePort——埠又醜(30080)、一個服務佔一個、不能用網址分流、也不能收 TLS。你想要的是:對外就開 80/443,靠 `shop.com`、`site.com/blog` 這種**網址**路由。

## 心智模型

- Ingress **站在 Service 前面**,不取代它。每個後端還是要有自己的 Service。
- 它看 HTTP 請求的 **host(網域)和 path(路徑)**,決定轉給哪個 Service → 這是 **L7(應用層)** 路由。
- 可以在這一層**集中收 TLS(HTTPS)**,後端之間走乾淨 HTTP。

```
外部請求 → Ingress(看 host/path、收 TLS) → Service → Pod
```

## 一件事看穿全局

**資源 vs 控制器**:`Ingress` 這個物件只是「規則」,要有一個 **Ingress Controller**(真正的反向代理)去執行它。沒裝 controller,規則就是廢紙。你的 k3s **內建 Traefik** 當 controller,所以開箱能用。

## 一句話記住

**同一個入口(80/443),靠網址分流到不同 Service。** Service 是水管,Ingress 是大門。

---
🔧 動手做:[notes/06-ingress.md](../notes/06-ingress.md)
← [05 Service](05-services.md)｜→ [07 ConfigMap & Secret](07-config.md)
