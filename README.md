# K8S Study Notes

Kubernetes learning notes, written while working through a local cluster.

## How to use this repo

Two layers, meant to be read together:

- **[`concepts/`](concepts/) — 概念篇(中文,先讀)**: short "grab the idea first" cards,
  one per lesson, plus two big-picture overviews. Start here to build the mental
  model before touching commands.
- **[`notes/`](notes/) — hands-on lessons (English)**: each lesson walks through the
  commands with real output (Part 1), explains how it works (Part 2), and ends with a
  command reference (Part 3). (Chinese `.zh.md` versions of each lesson are being added
  alongside.)

**Recommended path for a beginner:**

> 讀 [`concepts/`](concepts/) 的總覽(10 分鐘)→ 每一課先讀概念卡、再做對應的 `notes/` hands-on。

New here and no cluster yet? Start with
[Setting Up a Practice Cluster](notes/00-environment.md).

## Lessons

Read in order. Each row links the hands-on lesson; the matching Chinese concept card
is in [`concepts/`](concepts/).

| # | Lesson | What you get out of it |
| --- | --- | --- |
| 00 | [Setting Up a Practice Cluster](notes/00-environment.md) | A working k3s cluster in WSL2 you can break freely |
| 01 | [Architecture Overview](notes/01-architecture.md) | What the control plane and nodes do, and what happens after `kubectl apply` |
| 02 | [kubectl](notes/02-kubectl.md) | The command grammar, output formats, and how to answer your own questions |
| 03 | [Pods](notes/03-pods.md) | The smallest schedulable unit — and where self-healing stops |
| 04 | [Deployments](notes/04-deployments.md) | An owner that recreates, scales, and version-updates your Pods |
| 05 | [Services](notes/05-services.md) | A stable address and load balancing in front of churning Pods |
| 06 | [Ingress](notes/06-ingress.md) | One entry point routing by host and path to several Services |
| 07 | [ConfigMaps & Secrets](notes/07-config.md) | Feed configuration and credentials into Pods without rebuilding the image |

## Reference

Look these up as needed.

- [Object Reference](notes/objects.md) — every object type, what it is for, and the ownership chain
- [Troubleshooting](notes/troubleshooting.md) — a debugging method, then symptom-by-symptom fixes

## Practice files

- [`playground/`](playground/) — manifests used in the lessons

## About

Personal notes, updated as I learn. Hands-on lessons in [`notes/`](notes/) are
written in English (Chinese `.zh.md` versions are being added alongside); the concept
cards in [`concepts/`](concepts/) are in Chinese.

## License

MIT
