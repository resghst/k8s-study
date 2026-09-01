# K8S Study Notes

Kubernetes learning notes, written while working through a local cluster.

Each lesson is structured the same way: **Part 1 walks you through the commands
with their expected output**, **Part 2 explains how it works**, and **Part 3 is a
command reference** to come back to.

## Lessons

Read in order.

| # | Lesson | What you get out of it |
| --- | --- | --- |
| 00 | [Setting Up a Practice Cluster](notes/00-environment.md) | A working k3s cluster in WSL2 you can break freely |
| 01 | [Architecture Overview](notes/01-architecture.md) | What the control plane and nodes do, and what happens after `kubectl apply` |
| 02 | [kubectl](notes/02-kubectl.md) | The command grammar, output formats, and how to answer your own questions |
| 03 | [Pods](notes/03-pods.md) | The smallest schedulable unit — and where self-healing stops |
| 04 | [Deployments](notes/04-deployments.md) | An owner that recreates, scales, and version-updates your Pods |
| 05 | [Services](notes/05-services.md) | A stable address and load balancing in front of churning Pods |
| 06 | [Ingress](notes/06-ingress.md) | One entry point routing by host and path to several Services |

## Reference

Look these up as needed.

- [Object Reference](notes/objects.md) — every object type, what it is for, and the ownership chain
- [Troubleshooting](notes/troubleshooting.md) — a debugging method, then symptom-by-symptom fixes

## Practice files

- [`playground/`](playground/) — manifests used in the lessons

## About

Personal notes, updated as I learn. Note content is written in English.

## License

MIT
