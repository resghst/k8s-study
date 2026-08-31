# Core Concepts

## Pod

A Pod is the smallest deployable unit in Kubernetes. It holds one or more
containers that share the same network namespace and can share storage.

## ReplicaSet

A ReplicaSet keeps a specified number of Pod replicas running. You usually do not
create one directly — a Deployment manages it for you.

## Deployment

A Deployment provides declarative updates with rolling updates and rollback. In
practice, most stateless services are managed through a Deployment.

## Service

A Service gives a stable access point for a set of Pods. Common types:

- `ClusterIP` — reachable only inside the cluster (default)
- `NodePort` — exposed on a port of every node
- `LoadBalancer` — an external load balancer provisioned by the cloud provider
