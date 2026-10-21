---
title: CA Certificates & Trust Stores
---

## What a certificate is

When a program connects to `https://api.example.com`, it needs proof that the server on the other end is the real one. The server sends a **certificate**: an ID card that says "I am api.example.com", signed by someone who vouches for it.

## The chain of trust

Nobody signs certificates for the whole internet alone. There is a hierarchy:

```
Root CA 
   │ signs
Intermediate CA
   │ signs
Leaf cert
```

The client checks the chain upward. Who signed the leaf? An intermediate. Who signed the intermediate? A root. Is that root on my trusted list? If yes, the connection is trusted.

The important consequence: **any intermediate signed by a trusted root can sign a certificate for any domain.** The chain does not say "this office may only issue passports for example.com". An intermediate is as powerful as the root above it.

## The trust store

The list of trusted roots is the **trust store**. Usually it is a plain text file of concatenated PEM certificates.

The catch is that there is no single trust store. Every runtime picks its own:

| Runtime | Where it gets trusted roots | Override |
|---|---|---|
| Debian / Ubuntu / Alpine | `/etc/ssl/certs/ca-certificates.crt` from the `ca-certificates` package | `update-ca-certificates`, `/etc/ca-certificates.conf` |
| RHEL / Rocky | `/etc/pki/ca-trust/` | `update-ca-trust`, `source/blacklist/` |
| Go (Linux) | reads the OS bundle file **and** every file in `/etc/ssl/certs` | `SSL_CERT_FILE` + `SSL_CERT_DIR` |
| OpenSSL-based tools | OS bundle | `SSL_CERT_FILE` / `SSL_CERT_DIR` |
| Java | its own `cacerts` keystore | `-Djavax.net.ssl.trustStore` |
| Python `requests` | bundled `certifi` package, not the OS | `REQUESTS_CA_BUNDLE` |
| Python `grpcio` | its own bundled `roots.pem`, ignores certifi and the OS | `GRPC_DEFAULT_SSL_ROOTS_FILE_PATH` |
| Node.js | Mozilla roots compiled into the binary | `--use-openssl-ca` + `SSL_CERT_FILE` (`NODE_EXTRA_CA_CERTS` can only *add*) |
| macOS | Keychain, plus Homebrew's own `ca-certificates` / OpenSSL | `brew upgrade ca-certificates` |
| Dev toolchains (mise etc.) | sometimes ship their own list | tool update |

So "update the CA certs" is never one action. A container with a Go binary, a Python sidecar using gRPC and a Node script has three separate trust stores.

Go gotcha: setting only `SSL_CERT_FILE` is not enough. Go still loads every file in `/etc/ssl/certs`, which usually includes the full bundle again. Set `SSL_CERT_DIR` too.

## Revocation: why browsers and servers differ

If an intermediate goes bad, it can be **revoked** (put on a banned list).

- **Browsers** ship their own banned lists, updated constantly (Chrome CRLSet, Firefox OneCRL). They block bad intermediates within hours.
- **Server runtimes** (Go, Node, Python, OpenSSL) do no online revocation checking by default. They ask one question only: does this chain end at a trusted root?

So a revoked intermediate is dead in your browser and still fully trusted by your backend services and CI runners.

## Case study: compromised intermediate CA (eMudhra, October 2026)

### What happened

The private keys of eight eMudhra intermediate CAs (`EM DV/OV/EV TLS CA` and `EM OV CS CA`, generations G3B and G3C) were generated incorrectly, so anyone could recover them from the public certificates. Details are on the public disclosure page: https://emudhra.we-have-a-problem.com/

With the private key (the "stamp"), anyone can sign a valid-looking certificate for any domain, and it chains to a real, trusted root:

- `emSign Root TLS CA - G3` / `emSign Root CS CA - G3` (direct issuers), cross-signed by `emSign ECC Root CA - G3`
- `AAA Certificate Services` (Sectigo cross-signs of the same intermediates)

The attack this enables is **man-in-the-middle on outbound calls**: someone on the network path presents a forged certificate for an API your program calls, your program accepts it, and the attacker reads or changes the traffic. The exposure is when *your* program is the client checking someone else's certificate.

Browsers blocked the intermediates within days through their revocation lists. Server runtimes did not, because they do not check revocation.

### Two ways to fix it

| Fix | How | Where it lives |
|---|---|---|
| A. Patch the trust store | Remove the roots the bad intermediates chain to, so the chain no longer ends at a trusted root | The OS / container base image |
| B. Patch the runtime | Runtime ships a hard-coded blocklist of the bad intermediates' public key hashes | The compiled binary |

Fix A is blunt: other legitimate intermediates also chain to those roots, so removing a root can break valid sites. Fix B is precise, but each runtime has to ship it.

Fix B status at the time:

- **Go:** 1.26.10+ and 1.27.3+ (CVE-2026-111943, golang/go#82345). The blocklist is `distrustedSPKIHashes` in `crypto/x509/verify.go`: 8 SHA-256 hashes of the intermediates' SubjectPublicKeyInfo. Earlier patch releases of the same minor (for example 1.26.8) still accept the chains. You need the exact patch version.
- **Node.js:** the same 8 hashes, checked in a cert-verify callback and reported as `X509_V_ERR_CERT_REVOKED` ([nodejs/node#66871](https://github.com/nodejs/node/pull/66871)). Merged, but not in a release at the time.
- **Distro packages:** patched `ca-certificates` versions landed at different times per distro (for example Debian `20250419~deb12u2` / `20250419+deb13u1`, Alpine `20261020-r0`, Chainguard `20260909-r3`). The disclosure page keeps a full table.

For old runtimes that will not get fix B, the stopgap is fix A by hand, for example on Debian-based images:

```bash
sed -i \
  -e 's|^mozilla/emSign_ECC_Root_CA_-_G3.crt|!&|' \
  -e 's|^mozilla/Comodo_AAA_Services_root.crt|!&|' \
  /etc/ca-certificates.conf && update-ca-certificates
```

A `!` prefix in `/etc/ca-certificates.conf` marks the cert as distrusted. Test external calls afterwards, because removing a widely used root can break chains you rely on.

### Testing a trust store against the real attack

Checking root names is not enough, because cross-signs give the same intermediate several paths. The disclosure page publishes `chains.zip`: the 12 chains its test hosts serve (3 kinds x 2 generations x 2 roots), leaf first. Verify each one against the bundle you care about:

```bash
openssl verify -CAfile /path/to/ca-certificates.crt -untrusted chain.pem chain.pem
```

`OK` means the bundle still trusts a compromised intermediate. To pull a bundle out of a container image without running it:

```bash
crane export --platform linux/amd64 IMAGE - | tar -xO etc/ssl/certs/ca-certificates.crt > bundle.pem
```

Results at the time (out of 12 chains trusted):

| Image | Trusted |
|---|---|
| `gcr.io/distroless/static-debian13`, `base-debian13`, `static:nonroot` | 0 |
| `gcr.io/distroless/static-debian12` | 12 |
| `cgr.dev/chainguard/static` | 0 |
| `golang:1.26.10` / `-trixie` / `-bookworm`, `golang:1.27.3` | 0 |
| `golang:1.26.10-alpine` | 6 |

Two lessons from that table. The `debian12` and `debian13` variants of the same distroless image differ completely, so pick the variant on purpose. And a version number alone tells you little: distroless debian13 shipped an older `ca-certificates` (`20250419`) but was clean, because that package never contained the two affected roots.

The runtime fix is independent of the bundle. A small Go program calling `x509.Certificate.Verify` with the unpatched debian12 bundle accepted all 12 chains on go1.26.8 and rejected all 12 on go1.26.10.

### Why a fix takes a rebuild

A container image is a frozen snapshot. For a typical Go image:

```
compile     binary built with the Go version in go.mod     <- fix B lives here
layer       binary placed on top of a base image           <- fix A lives here
deploy      the cluster runs that exact image
```

Running containers keep the old binary and the old bundle until the image is rebuilt and redeployed. Base images pinned by digest do not move on their own, and build systems that pass the Go version as a build argument (for example `--build-arg GOVERSION=...`) need that value bumped too. Official `golang` images set `GOTOOLCHAIN=local`, so a newer `go` directive in `go.mod` fails the build instead of downloading the toolchain.

## Takeaways

- An intermediate CA is as powerful as its root. One leaked intermediate key makes every domain forgeable.
- Browsers being safe tells you nothing about your servers. Server runtimes do not check revocation.
- Inventory trust stores per runtime, not per machine. "We updated ca-certificates" does not cover Node, Java or gRPC Python.
- Test against real chains, not root names or package versions.
- A fix is only live after a rebuild plus redeploy. Know where your Go version and base image are pinned before you need to move them all at once.
