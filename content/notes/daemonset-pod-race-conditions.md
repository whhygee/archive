---
title: DaemonSet Pod Race Conditions
---

## The Problem

When a DaemonSet (e.g. mitmproxy) runs alongside regular pods (e.g. CI runners), there's a race condition: the runner pod can start before the DaemonSet pod is ready on that node. The runner then tries to use the proxy, gets "connection refused," and either fails or bypasses the proxy entirely.

This commonly happens during **node scale-up** — a new node joins the cluster, and both the DaemonSet pod and a runner pod get scheduled simultaneously. The runner may start first.

## Solutions

### 1. Init Container (Simple)

Add an init container to the runner pod that blocks until the proxy port is reachable. The main containers won't start until this passes.

```yaml
initContainers:
- name: wait-for-proxy
  image: busybox:1.36
  env:
  - name: NODE_IP
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
  command:
  - sh
  - -c
  - |
    until nc -z ${NODE_IP} 18080; do
      echo "Waiting for proxy on ${NODE_IP}:18080..."
      sleep 2
    done
```

`status.hostIP` is the **Downward API** — Kubernetes injects the node's IP into the pod as an environment variable. Since the DaemonSet uses `hostPort: 18080`, the proxy is reachable at `NODE_IP:18080` from any pod on that node.

**Pros:** Self-contained, no extra RBAC needed, easy to understand.
**Cons:** Adds startup latency (the poll interval). If the DaemonSet crashes permanently, the runner hangs forever (add a timeout to mitigate).

With a timeout to avoid hanging:
```yaml
  command:
  - sh
  - -c
  - |
    TIMEOUT=60
    ELAPSED=0
    until nc -z ${NODE_IP} 18080; do
      echo "Waiting for proxy on ${NODE_IP}:18080... (${ELAPSED}s)"
      sleep 2
      ELAPSED=$((ELAPSED + 2))
      if [ $ELAPSED -ge $TIMEOUT ]; then
        echo "Proxy not ready after ${TIMEOUT}s, proceeding without it"
        exit 0
      fi
    done
```

### 2. Taint/Toleration (Scheduling-Level)

Prevent the runner from being scheduled at all until the DaemonSet pod is running.

1. **Taint the node pool:** `proxy-not-ready=true:NoSchedule`
2. **DaemonSet tolerates the taint** (so it schedules anyway)
3. **DaemonSet pod removes the taint** once ready (via a startup script that calls `kubectl taint nodes`)
4. **Runner pods don't tolerate it** — they can't schedule until the taint is gone

```yaml
# On the DaemonSet pod spec
tolerations:
- key: proxy-not-ready
  operator: Equal
  value: "true"
  effect: NoSchedule

# Lifecycle hook or startup script in the DaemonSet
lifecycle:
  postStart:
    exec:
      command:
      - sh
      - -c
      - kubectl taint nodes ${NODE_NAME} proxy-not-ready- || true
```

**Pros:** Prevents the race entirely at the scheduler level — runner pods never start on a node without a ready proxy.
**Cons:** Requires RBAC for the DaemonSet to patch node taints. If the DaemonSet crashes, the taint returns and blocks all new pods on that node until it recovers. More moving parts.

### 3. internalTrafficPolicy: Local (Complementary, Not a Fix)

If the proxy is exposed as a Kubernetes Service, you can set `internalTrafficPolicy: Local`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mitmproxy
spec:
  internalTrafficPolicy: Local
  selector:
    app: mitmproxy
  ports:
  - port: 18080
```

This tells kube-proxy: only route to proxy pods on the **same node**. Without it, traffic might hop to a proxy on a different node (defeating per-node observability).

**Important:** This does NOT solve the race condition. If the local DaemonSet pod isn't ready yet, there are zero eligible endpoints on that node, and the connection fails. Unlike the default policy where kube-proxy would fall back to a proxy on another node, `Local` has no fallback. It's useful once the DaemonSet is stable, but pair it with option 1 or 2 to handle startup.

## Real-World Example: Istio Sidecar Injection

Istio solves the exact same race condition at scale. Every pod in an Istio mesh gets an Envoy sidecar proxy injected automatically. The problem: the app container might start sending traffic before the Envoy sidecar is ready to intercept it. Traffic would bypass the mesh entirely.

### How Istio handles it

Istio injects an **init container** called `istio-init` into every pod. This init container runs before any app containers start and sets up **iptables rules** that redirect all inbound and outbound traffic through the Envoy sidecar.

```
Pod startup order:
1. istio-init (init container) → sets up iptables redirect rules
2. istio-proxy (sidecar)      → Envoy starts, listens on the redirected ports
3. app container              → all traffic is already being captured by iptables
```

The iptables rules work at the kernel level — they redirect traffic regardless of whether the app "knows" about the proxy. So even if the app starts before Envoy is fully ready, the traffic doesn't escape. It just queues in the kernel until Envoy accepts the connection.

```
# Simplified version of what istio-init does:
iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-port 15001   # outbound → Envoy
iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-port 15006  # inbound → Envoy
```

### Why Istio uses init containers instead of taints

- **Scale:** Istio runs on every pod in the mesh (thousands). Tainting every node and managing taint lifecycle would be operationally painful.
- **Transparency:** Apps don't need to know about the proxy. The iptables redirect is invisible to the app — no `http_proxy` env vars needed.
- **No DaemonSet dependency:** The Envoy sidecar runs as a container in the same pod (not a DaemonSet), so there's no cross-pod race. The init container and sidecar are co-scheduled.

### Key difference from our mitmproxy setup

Istio's sidecar runs **inside the same pod** as the app — they're always co-scheduled. Our mitmproxy runs as a **DaemonSet** (separate pod on the same node). That's why Istio's init container can set up iptables with certainty (the sidecar is right there), while our init container has to poll an external port and hope the DaemonSet pod is ready.

| | Istio | Our mitmproxy |
|---|---|---|
| Proxy location | Same pod (sidecar) | Same node (DaemonSet) |
| Traffic capture | iptables redirect (kernel) | http_proxy env var (app-level) |
| Init container role | Set up iptables rules | Poll until proxy port is reachable |
| Race risk | Minimal (co-scheduled) | Real (separate pod scheduling) |

### holdApplicationUntilProxyStarts

Even with the init container, there's still a brief window where the app container starts before Envoy is fully ready. Istio added `holdApplicationUntilProxyStarts: true` to handle this — it makes the sidecar container start first and delays the app container until Envoy's readiness probe passes. This is essentially the same idea as our init container polling approach, but built into Istio's injection logic.

## Recommendation

For a **debugging/observability proxy** (like mitmproxy on lab/dev): use the **init container** approach. It's simple, self-contained, and the timeout fallback means runners aren't permanently blocked if the proxy has issues.

For a **critical proxy in production** (where traffic MUST go through the proxy): use the **taint/toleration** approach. The stronger guarantee is worth the operational complexity.

Either way, add `internalTrafficPolicy: Local` on the Service to keep traffic node-local.

## Related

- [[notes/proxies-and-tls-termination|Proxies & TLS Termination]] — how the proxy itself works
- [[notes/kubernetes|Kubernetes Concepts]] — DaemonSets, taints, node pools
- [[notes/cloud-nat-and-vpc-networking|Cloud NAT & VPC Networking]] — why we proxy GitHub traffic
