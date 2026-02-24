---
title: Kubernetes Concepts
---

Core Kubernetes concepts referenced across the archive.

## DaemonSet

A regular Deployment says "run N copies of this pod somewhere." A DaemonSet says "run exactly one copy on every node" (or every node matching a label selector).

When nodes are added to the cluster, the DaemonSet automatically puts a pod on them. When removed, the pod goes too. Used for things that need to run on every machine: log collectors, monitoring agents, network proxies.

For the proxy use case (see [[notes/proxies-and-tls-termination|Proxies & TLS Termination]]), a DaemonSet means each node gets its own proxy instance. Runners access it at the node's IP, keeping traffic local to the node and avoiding a single bottleneck.

## Node Pools

A GKE cluster can have multiple **node pools** — groups of VMs with different configurations (machine type, disk, labels, taints). You might have a "default" pool for general workloads and a "critical" pool for important CI jobs.

## nodeSelector and Tolerations

To control which pods land on which nodes:

- **nodeSelector** — a pod-level setting that says "only schedule me on nodes with this label." E.g., `nodeSelector: {pool: critical}` means the pod only runs on nodes labeled `pool=critical`.
- **Taints and tolerations** — taints are the opposite. A **taint** on a node says "don't schedule anything here unless it explicitly tolerates me." Think of it like a "keep out" sign. A **toleration** on a pod says "I'm allowed past that sign." This prevents random workloads from landing on special nodes.
  - `operator: Equal` (default) — matches a specific taint key+value. Need one toleration per value.
  - `operator: Exists` — matches a taint key regardless of value. One toleration covers all values of that key. E.g., `{key: runner-type, operator: Exists, effect: NoSchedule}` tolerates `runner-type=android`, `runner-type=image-cached`, etc.

### Why this matters for NAT

Different node pools can use different subnets. Different subnets can have different NAT rules and IPs. So by steering workloads to specific node pools, you control which NAT IPs they use — separating critical traffic from noisy traffic. See [[notes/cloud-nat-and-vpc-networking|Cloud NAT & VPC Networking]] for details.
