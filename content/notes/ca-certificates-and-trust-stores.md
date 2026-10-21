---
title: CA Certificates & Trust Stores
---

> Case study: [[notes/emudhra-ca-compromise-case-study|eMudhra CA Compromise (October 2026)]]. A real incident that shows every concept on this page.

Collapsed boxes marked "Details" hold the deep technical parts. You can skip them and still follow the page.

## What a certificate is

When a program connects to `https://api.example.com`, it needs proof that the server on the other end is the real one. The server sends a **certificate**: an ID card that says "I am api.example.com", signed by someone who vouches for it. That someone is a **CA** (certificate authority).

## The chain of trust

Nobody signs certificates for the whole internet alone. There is a hierarchy:

```
Root CA
   │ signs
Intermediate CA
   │ signs
Leaf cert (api.example.com)
```

The client checks the chain upward. Who signed the leaf? An intermediate. Who signed the intermediate? A root. Is that root on my trusted list? If yes, the connection is trusted.

The important consequence: **any intermediate signed by a trusted root can sign a certificate for any domain.** The chain does not say "this office may only issue passports for example.com". An intermediate is as powerful as the root above it.

## How chaining works

A certificate has two parts:

1. **The content**: "this public key belongs to `api.example.com`", plus valid-from and valid-until dates and a few rules.
2. **A signature over the content**, made with the issuer's private key.

To check a certificate, you test the signature with the issuer's public key. If it passes, the issuer really signed it and nobody changed the content since. A chain repeats that check one link at a time:

```
leaf cert          checked with  -> intermediate's public key
intermediate cert  checked with  -> root's public key
root cert          not checked   -> trusted because it is in your trust store
```

The root is the only link that math does not prove. You trust it because your OS, browser or runtime put it on your machine.

So "being allowed to chain up to a root" means one thing: you hold a private key, and a trusted root (directly or through other CAs) signed a certificate for that key with the flag "this key may sign other certificates".

Besides the signature, the client also checks that each cert is in its date range, that each parent has the "may sign certificates" flag, and that the leaf names the hostname you connected to. **No check asks whether this CA should be signing for this domain.** If the chain checks out, any CA can vouch for any domain.

> [!info]- Details: what is inside a certificate
> The content part is called `tbsCertificate` ("to be signed"). The file is DER-encoded ASN.1 (a binary format) with three parts: the content, the signature algorithm name, and the signature bytes. Only the content is signed.
>
> | Field | What it says |
> |---|---|
> | `subject` | Who this cert is for |
> | `issuer` | Who signed it. Must match the `subject` of the cert one level up |
> | `subjectPublicKeyInfo` (SPKI) | The public key of the subject |
> | `notBefore` / `notAfter` | Valid from / until |
> | Basic Constraints `cA` | `TRUE` = this key may sign other certificates. Leaves have `FALSE` |
> | `pathLenConstraint` | How many more CA levels may exist below this one |
> | Key Usage `keyCertSign` | Another "may sign certs" flag. CAs need it |
> | Extended Key Usage | What the leaf may be used for, for example `serverAuth` for TLS servers |
> | Subject Alternative Name (SAN) | The hostnames the leaf is valid for |
> | Name Constraints | Optional limit on which domains a CA may sign for. Rare in public CAs |
> | SKI / AKI | Key IDs that help find the right parent cert quickly |
>
> Signing: the issuer hashes the content (for example SHA-256) and signs the hash with its private key. With ECDSA the result is two numbers, `(r, s)`. With RSA it is the padded hash raised to the private exponent.

> [!info]- Details: the full list of client checks (RFC 5280)
> The server sends its leaf and the intermediates, usually without the root. If links are missing, the client looks in its own store. Windows and macOS can also download missing parents from a URL in the cert.
>
> The client walks from the root down to the leaf ([RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280#section-6) section 6). For each cert:
>
> 1. The signature is valid, checked with the parent's public key.
> 2. Today is between `notBefore` and `notAfter`.
> 3. `issuer` equals the parent's `subject`.
> 4. It is not revoked (if the client checks revocation at all, see below).
> 5. Every CA cert has `cA = TRUE` and `keyCertSign`, and the chain is not deeper than `pathLenConstraint` allows.
> 6. If a parent has Name Constraints, the names below it stay inside them.
> 7. An unknown extension marked "critical" means reject.
>
> Then the leaf: the hostname is in the SAN, and Extended Key Usage allows `serverAuth`. Chrome and Safari also require proof that the cert is in public Certificate Transparency (CT) logs.
>
> The root is self-signed (issuer and subject are the same). The client does not check the root's own signature. It only needs the root's public key.

### Cross-signing

A CA can get the same public key signed by two different parents. A leaf signed by that key then has two valid paths to a root.

New CAs use this because a new root takes years to reach old devices. An established CA signs the new CA's key, so old devices still find a path. Let's Encrypt did this with IdenTrust.

The catch: revoking one path does not close the other. Every cross-signed copy has to be revoked on its own.

## Roots vs intermediates

Root stores hold roots only, never intermediates. A CA usually has one RSA root and one ECC root per business line, so a root program can distrust one algorithm or business line without the others.

The root key is kept **offline**: in a hardware security module (HSM) that is powered off, has no network, and sits in a safe. It comes out only for a **key ceremony**, a witnessed, scripted session. Nobody, including the CA's own staff, ever sees the key bytes.

At a ceremony, the root signs a new intermediate. The intermediate's key then lives in an online HSM and signs customer certs day to day. The root key is used only a few times in its life: at creation, for each new intermediate, and to reissue its revocation list.

That is the reason for two tiers. Online keys are exposed, so you can afford to lose and revoke them. A stolen root key cannot be fixed, except by removing the root from every store.

> [!info]- Details: ceremony steps
> 1. The intermediate's key pair is generated in an online HSM.
> 2. That HSM produces a CSR (Certificate Signing Request): subject name, public key, requested extensions, signed with the new private key to prove possession.
> 3. The CSR goes on removable media into the ceremony room. The root HSM powers on. A quorum of key holders inserts smart cards to unlock the root key.
> 4. The operator builds the intermediate's content (issuer = root, subject and public key from the CSR, `CA:TRUE`, dates, extensions). The root HSM signs it. The root key never leaves the device.
> 5. The signed intermediate is copied out. The root HSM powers down and goes back in the safe.

## How a CA gets into root stores

There is no central body. Each **root program** decides on its own: Mozilla, Chrome, Apple, Microsoft, plus smaller ones such as Java and Android.

1. Build a CA that follows the CA/Browser Forum Baseline Requirements (the industry rulebook).
2. Pass an annual audit from an accredited auditor.
3. Apply to each root program. Reviewers read your policies line by line, and the community can object. This takes 1 to 3 years and a lot of money.
4. Stay included: report incidents in public, revoke bad certs on time, do not mis-issue.

A government license gives nothing here. For example, India's CCA (Controller of Certifying Authorities) root is in no browser.

For internal infrastructure you skip all of this. Generate your own root and push it to your machines' trust stores. You are then your own root program.

> [!info]- Details: what the requirements and audits involve
> - Baseline Requirements: HSM-backed keys, offline root, audited key ceremonies with witnesses and scripts, a published CP/CPS (certificate policy and practice statement), CT logging, approved domain-validation methods.
> - Audits: WebTrust for CAs or ETSI EN 319 411.
> - Mozilla's process is public, on Bugzilla and the CCADB. Reviewers also inspect test certificates.
> - Programs now cap root lifetimes and expect rotation.
> - Internal roots get pushed through MDM, config management, or base images.

## Choosing a CA

For TLS server certs the choice is mostly about workflow, not trust:

- **Trust** does not differentiate. Any CA in the Mozilla, Apple, Microsoft and Chrome programs works in every browser.
- **Automation**: if you can use ACME (the automatic renewal protocol), Let's Encrypt (or Google Trust Services, ZeroSSL) is the default. Certificate lifetimes are shrinking toward 47 days, so manual renewal stops working.
- **Validation level**: DV (domain validated) is enough for almost everything. Use OV/EV only if compliance or a customer requires it. Browsers no longer show EV specially.
- **Internal services**: run your own CA (step-ca, Vault PKI, GCP CAS) and pay no public CA.
- **Paid CAs** (DigiCert, Sectigo, GlobalSign) earn their fee with support, management consoles, code signing, S/MIME, and contracts with SLAs.
- **Track record**: distrusted CAs (Symantec, Entrust's TLS business) show that big does not mean safe.

## The trust store

The list of trusted roots is the **trust store**. Usually it is a plain text file of PEM certificates joined together.

The catch: there is no single trust store. Every runtime picks its own, so "update the CA certs" is never one action. A container with a Go binary, a Python sidecar using gRPC and a Node script has three separate trust stores.

| Runtime | Where it gets trusted roots | Override |
|---|---|---|
| Debian / Ubuntu / Alpine | `/etc/ssl/certs/ca-certificates.crt` from the `ca-certificates` package | `update-ca-certificates`, `/etc/ca-certificates.conf` |
| RHEL / Rocky | `/etc/pki/ca-trust/` | `update-ca-trust`, `source/blacklist/` |
| Go (Linux) | the OS bundle file **and** every file in `/etc/ssl/certs` | `SSL_CERT_FILE` + `SSL_CERT_DIR` |
| OpenSSL-based tools | OS bundle | `SSL_CERT_FILE` / `SSL_CERT_DIR` |
| Java | its own `cacerts` keystore | `-Djavax.net.ssl.trustStore` |
| Python `requests` | bundled `certifi` package, not the OS | `REQUESTS_CA_BUNDLE` |
| Python `grpcio` | its own bundled `roots.pem`, ignores certifi and the OS | `GRPC_DEFAULT_SSL_ROOTS_FILE_PATH` |
| Node.js | Mozilla roots compiled into the binary | `--use-openssl-ca` + `SSL_CERT_FILE` (`NODE_EXTRA_CA_CERTS` can only *add*) |
| macOS | Keychain, plus Homebrew's own `ca-certificates` / OpenSSL | `brew upgrade ca-certificates` |

Go gotcha: setting only `SSL_CERT_FILE` is not enough. Go still loads every file in `/etc/ssl/certs`, which usually includes the full bundle again. Set `SSL_CERT_DIR` too.

Where the lists come from: Firefox uses Mozilla's list (built into NSS). Chrome has its own Chrome Root Store. Apple and Microsoft ship theirs with OS updates. Linux distros, Python `certifi` and Node copy Mozilla's list.

## Revocation: why browsers and servers differ

If an intermediate goes bad, it can be **revoked** (put on a banned list).

- **Browsers** get their banned lists pushed to them constantly (Chrome CRLSet, Firefox OneCRL), with no browser update. They block bad intermediates within hours.
- **Server runtimes** (Go, Node, Python, OpenSSL) do no online revocation checking by default. They ask one question only: does this chain end at a trusted root?
- **Copies of the root list** (distro packages, certifi, base images) change only when someone rebuilds them.

So a revoked intermediate is dead in your browser and still fully trusted by your backend services and CI runners. That is why, in the eMudhra case, Go and Node had to hardcode a block into their own code.

## Java

Java has its own trust store and no push channel. The JDK ships `$JAVA_HOME/lib/security/cacerts`, maintained by Oracle's root program, independent of Mozilla. By default Java also does no revocation checking, so a stock JVM trusts every root in that file until the file changes.

Some JVMs use the OS store instead and get the OS fix for free. For base images, find out whether `cacerts` comes from the OS bundle or the JDK tarball. That decides whether a fix arrives on its own.

> [!info]- Details: Java settings and manual root removal
> - Revocation checking is off unless you set `com.sun.net.ssl.checkRevocation=true` and related properties.
> - JVMs that use the OS store: `-Djavax.net.ssl.trustStoreType=Windows-ROOT`, or Debian/Ubuntu `ca-certificates-java`, which rebuilds `cacerts` from `/etc/ssl/certs`.
> - To remove a root by hand, do it in the base image, then rebuild downstream images:
>
> ```bash
> keytool -list -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit | grep -i emsign
> keytool -delete -alias <alias> -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit
> ```

## Takeaways

- An intermediate CA is as powerful as its root. One leaked intermediate key makes every domain forgeable.
- Browsers being safe tells you nothing about your servers. Server runtimes do not check revocation.
- Inventory trust stores per runtime, not per machine. "We updated ca-certificates" does not cover Node, Java or gRPC Python.
- Revoke every cross-signed copy of a bad key, not only one path.
- Test against real chains, not root names or package versions.
- A fix is only live after a rebuild plus redeploy. Know where your Go version and base image are pinned before you need to move them all at once.
