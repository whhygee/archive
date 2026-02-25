---
title: Proxies & TLS Termination
---

## What is a Proxy

A proxy is a middleman that sits between a client and a server. Instead of the client talking directly to the server, it talks to the proxy, and the proxy forwards the request.

```
Client → Proxy → Server
Client ← Proxy ← Server
```

### Forward Proxy vs Reverse Proxy

- **Forward proxy** — sits in front of clients. The client knows about the proxy and sends traffic through it. Used for: controlling outbound access, caching, observability. Example: Envoy or mitmproxy in front of CI runners.
- **Reverse proxy** — sits in front of servers. The client doesn't know the proxy exists — it thinks it's talking to the real server. Used for: load balancing, SSL offloading, routing. Example: nginx in front of a web app.

## Network Layers (L3, L4, L7)

Networking is described in layers. The ones that matter most:

- **L3 (Network)** — IP addresses. "This packet goes from 10.0.1.5 to 140.82.113.3." Cloud NAT operates here.
- **L4 (Transport)** — TCP/UDP ports. "This is a TCP connection on port 443." Firewalls and basic load balancers operate here.
- **L7 (Application)** — HTTP, the actual request. "GET /org/some-repo with Authorization header and response was 401." Envoy, mitmproxy, nginx operate here.

The higher the layer, the more the proxy can see and do. Cloud NAT (L3/L4) can't tell you that GitHub returned a 401. An L7 proxy can.

## TLS and HTTPS

**TLS** (Transport Layer Security) is the encryption that turns HTTP into HTTPS. When you connect to `https://github.com`:

1. Your client and GitHub do a TLS handshake (exchange keys)
2. All traffic is encrypted end-to-end
3. No one in the middle can read the content

This is great for security, but it means a proxy in the middle also can't see the content — unless it "terminates" the TLS.

## TLS Termination

"Termination" means **where the encrypted tunnel ends**. Normally:

```
Client ────── encrypted tunnel ────── GitHub
               (ends at GitHub)
```

With TLS termination at a proxy:

```
Client ── tunnel 1 ── Proxy ── tunnel 2 ── GitHub
            (ends here)   (new tunnel starts)
```

The proxy decrypts the traffic (tunnel 1 ends), reads the plain HTTP, then opens a new encrypted connection to the real server (tunnel 2). This is also called **MITM** (man-in-the-middle) because the proxy is literally intercepting encrypted traffic.

### How the client trusts the proxy

Normally your client trusts GitHub because GitHub has a certificate signed by a well-known Certificate Authority (CA) like DigiCert. Your OS/browser ships with a list of trusted CAs.

When a proxy terminates TLS, it generates a **fake certificate** that says "I'm github.com" — but it's signed by the proxy's own CA, not DigiCert. For the client to accept this, you need to **inject the proxy's CA certificate** into the client's trust store.

There are two certs involved:

1. **CA cert (root)** — generated once, stays the same. This is what you install on clients.
2. **Per-site cert (leaf)** — generated on the fly for each destination (github.com, api.github.com, etc.). Signed by the CA cert.

The client trusts the CA cert, so it automatically trusts any leaf cert signed by it.

### Security implications

With TLS termination, the proxy has access to **everything** in plaintext:
- Full URLs
- Authorization headers (tokens, passwords)
- Request/response bodies

You can mask secrets in logs (e.g., redact `Authorization: Bearer xxx` before writing to disk), but the proxy process itself sees them in memory. Anyone who can exec into the proxy pod or dump its memory can read tokens.

For short-term debugging, this is usually an acceptable tradeoff. For long-term production, it's a risk surface.

## Envoy

Envoy is a high-performance, open-source L7 proxy written in C++. It's the proxy behind Istio and many service meshes. For the CI observability use case:

- Run as a **[[notes/kubernetes#DaemonSet|DaemonSet]]** (one pod per Kubernetes node)
- Runners on each node point to it via `HTTPS_PROXY` env var
- Envoy forwards traffic to GitHub while logging request/response metadata

Envoy config has four main concepts:

- **Listener** — the port Envoy listens on (e.g., 8080)
- **Route** — rules for where to forward traffic based on destination
- **Cluster** — a group of upstream servers (e.g., GitHub's IPs)
- **Filter** — processing steps applied to traffic (logging, header manipulation, secret masking via Lua scripts)

Envoy is production-grade and handles high throughput, but configuration is verbose YAML and filters require Lua. Better suited for long-term setups.

## mitmproxy

mitmproxy is a Python-based interactive HTTPS proxy built specifically for inspecting traffic. Compared to Envoy:

- **Easier setup** — runs out of the box, auto-generates CA certs
- **Python scripting** — write addons in Python to filter/mask/log requests (much simpler than Envoy's Lua)
- **Built-in UI** — `mitmweb` gives a browser interface to inspect requests live
- **Lower throughput** — single-threaded Python, fine for debugging, not for heavy production load

Setup in Kubernetes:
1. Deploy as a [[notes/kubernetes#DaemonSet|DaemonSet]]
2. Mount the CA cert as a Kubernetes Secret
3. Inject the CA cert into runner pods (env var or trust store)
4. Set `HTTPS_PROXY=http://<node-ip>:<port>` on runners
5. Write a Python addon to mask `Authorization` headers before logging

### Without TLS termination (tunnel mode)

Both Envoy and mitmproxy can run as a **CONNECT proxy** without terminating TLS. In this mode:
- The proxy sees the **destination hostname** (from SNI) and connection-level success/failure
- The proxy does **not** see HTTP status codes, URLs, or headers
- No CA cert injection needed — much simpler
- Less visibility, but enough if you just need "which host is this runner connecting to"

## Athens Proxy (Go Modules)

A different kind of proxy — not for general HTTP traffic, but specifically for Go module downloads. Athens caches Go modules so that `go mod download` doesn't hit GitHub directly every time.

Configured via `GOPROXY=http://athens-proxy.athens.svc.cluster.local:3000`. When Go needs a module:
1. Asks Athens first
2. If Athens has it cached, returns it immediately (no GitHub request)
3. If not, Athens fetches from GitHub, caches it, and returns it

This reduces unauthenticated requests to GitHub dramatically. The key gap: Athens only helps if the workflow sets the `GOPROXY` env var. Workflows that don't (like CodeQL's default setup) bypass Athens entirely and hit GitHub directly.

## See also

- [[notes/observability-hacks]] — practical techniques (docker binary wrapper, MITM proxy) for debugging CI traffic without full proxy infrastructure
