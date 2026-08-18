---
title: Pod Evictions & Resource Requests
---

## The two numbers on every container

Every container can declare two memory numbers:

- **request** — how much it says it needs. Kubernetes reserves this.
- **limit** — its hard ceiling. Go over it and the container is killed.

Both are optional. Leaving them out is risky, and the reasons why are the subject of this note.

## Requests do two different jobs

1. **Deciding who gets killed.** When a node runs out of memory, something has to go. Requests are how Kubernetes decides who was being reasonable and who wasn't.
2. **Deciding whether to add machines.** The autoscaler adds nodes when pods can't fit. "Fit" is calculated from requests only. Actual memory usage is invisible to it.

The second one is the trap. A pod that requests nothing fits *anywhere*, because it claims to need nothing. It is never stuck waiting -> the autoscaler is never told there's a problem -> no new machines arrive. The cluster packs pods onto a node that is running out of memory while the autoscaler sits idle, because as far as it knows demand is zero.

Requesting nothing does not mean "no opinion". It means "I need nothing", and Kubernetes believes you.

## Three tiers of protection

Your request and limit settings sort the pod into one of three tiers, which is the eviction order:

| Tier | What you set | Killed |
|---|---|---|
| Guaranteed | request and limit, equal | last |
| Burstable | a request, lower than the limit | middle |
| **BestEffort** | **nothing at all** | **first** |

Set nothing and you land in the bottom tier. Under memory pressure those pods are cleared out before anything else on the node.

## Killed for being over your request, not for being big

When Kubernetes picks victims it does not ask "who is using the most memory?" It asks **"who is furthest over what they asked for?"**

- Pod A asked for nothing and uses 20 GB. It is 20 GB over its request.
- Pod B asked for 50 GB and uses 40 GB. It is *under* its request.

Pod A gets killed. Pod B, using twice as much memory and the actual reason the node is struggling, is left alone. It asked for room, so it's entitled to it.

So **the pod that gets killed is often not the pod causing the problem.** Check what else is on the node and what it requested.

## Exit code 137 means two different things

`137` means the process was killed with the unblockable kill signal. It tells you the process was killed from outside and nothing more. Two different situations produce it:

| Reason | What happened | Whose fault |
|---|---|---|
| `OOMKilled` | the container went over **its own** limit | that container |
| `Evicted` | the **node** ran out and Kubernetes chose a victim | possibly another pod entirely |

`OOMKilled` means look at that container. `Evicted` means look at the whole node.

A container with **no limit set can never be `OOMKilled`**, having no ceiling to exceed. Its only way to die by memory is eviction, so the cause may be somebody else.

## Keeping workloads off each other's machines

To stop two kinds of work fighting over the same memory, put them on separate machines. That takes two settings working in opposite directions:

- **A label selector on your pod** pulls it onto the machines you want.
- **A taint on those machines** keeps everything else off. A taint is a "do not schedule here" mark; only pods that explicitly tolerate it may land.

You need both. The selector alone lets other pods wander in. The taint alone lets your pods wander out. Together they make a closed door.

This is also how you answer "did this crash affect users?" Memory pressure is a property of a **machine**, not of a namespace. Namespaces separate names and permissions and do nothing about memory. The question is only ever "were they on the same machine?", and taints are what let you answer it.
