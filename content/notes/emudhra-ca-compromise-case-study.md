---
title: eMudhra CA Compromise (October 2026)
---

> Concepts used here (chains, trust stores, cross-signs, root programs) are in [[notes/ca-certificates-and-trust-stores|CA Certificates & Trust Stores]].

Collapsed boxes marked "Details" hold the deep technical parts. You can skip them and still follow the page.

## Summary

Eight eMudhra intermediate CAs had private keys so weak that anyone could work them out from the public certificates. With those keys, anyone could make a trusted certificate for any domain. Browsers blocked them within days. Servers, containers and CI runners stayed exposed until their runtime or trust store was updated **and** the image was rebuilt. There is no evidence anyone abused the keys.

Full public disclosure: https://emudhra.we-have-a-problem.com/

## Who eMudhra is

eMudhra grew on India's regulatory structure, not the browser market.

- Founded in 2008 as a licensed Certifying Authority under India's IT Act 2000. It started with Digital Signature Certificates (DSCs), which Indian law requires for many official filings.
- India's largest licensed CA: 37.9% of the DSC market in FY21, up from 36.5% in FY20.
- IPO in May 2022, listed June 1, 2022.
- FY2026 revenue grew 35% (enterprise solutions +55%, trust services +32%). About half of the new revenue came from acquisitions, for example Cryptas International. International revenue was 64% of the total.
- Products include emSigner (e-signature workflows) and emCA (a PKI platform).

The Indian government license does not put a root in any browser. eMudhra went through the browser root programs separately for its public brand, emSign. For public TLS it is a small player.

Unverified (from chat research, no source found): over 100 million DSCs issued, emCA sold to governments such as Kenya, only Indian member of the Cloud Signature Consortium and CA/Browser Forum, the exact list of filings (GST, income tax, MCA/ROC, tenders, customs) that require DSCs.

Sources: [SBI Securities IPO note](https://sbisecurities.in/fileserver/research/reports/eMudhra_IPO_Note_19-05-2022.pdf), [Business Today](https://www.businesstoday.in/amp/markets/ipo-corner/story/emudhra-collects-rs-124-crore-from-anchor-investors-ahead-of-ipo-334327-2022-05-19), [ICRA](https://www.icra.in/Rating/GetRationalReportFilePdf?id=143969), [StockAnalysis](https://stockanalysis.com/quote/nse/EMUDHRA/financials/income-statement/).

## Why a small CA becomes a global incident

Every CA in a root store can sign a certificate for any domain. Being small does not matter. A key that chains to a trusted root can impersonate google.com to every device that trusts that root, which is almost every device. One bad key ceremony therefore becomes everyone's problem.

## What went wrong

A private key is supposed to be a huge random number, so large that guessing it is impossible. All eight eMudhra keys were numbers below 256. One key was literally the number 4. Anyone could try all 256 values in milliseconds and find the key.

- Eight intermediates: six for TLS (websites), two for Microsoft code signing (signing programs).
- They were created at a key ceremony on April 30, 2025. eMudhra's early finding: a modified key-generation script, which added a wrap/unwrap test, caused the weak keys. The investigation was still open.
- A researcher reported it on October 17, 2026, and proved it by signing the message "We have a problem here – Wayne" with each key.
- Mozilla (Dennis Jackson) found the keys on its own. Its report says AI agents found them on the first try.
- Scott Helme found key values 4, 56 and 115.

Nothing in normal certificate checking can catch this. The root signed these intermediates correctly at the ceremony, so their certificates are genuine. The keys were not stolen. They were guessable, which has the same effect as handing them out. The only defenses are blocking the intermediates or removing their root.

> [!info]- Details: the math
> The keys are P-384 ECDSA keys. The public key is Q = d·G, where d is the private number and G a fixed point on the curve. Security depends on d being random in a range of about 2³⁸⁴ values, so nobody can find d from Q. With d < 256, anyone computes d·G for d = 1 to 255 and compares with the published Q. With d, they can sign any certificate they write, and the signature check against the real intermediate passes.
>
> In ceremony terms, step 1 (generate the intermediate key pair) went wrong. Step 4 (root signs the intermediate) worked as designed, which is why clients accepted the chain.

Sources: [gbhackers](https://gbhackers.com/emudhra-security-incident-exposes-weak-private-keys/amp/), [Scott Helme](https://scotthelme.co.uk/eight-bits-of-trust-emudhras-ca-keys-cracked-in-milliseconds/), [Mozilla bug 2083449](https://bugzilla.mozilla.org/show_bug.cgi?id=2083449), [CSO Online](https://www.csoonline.com/article/4238338/google-mozilla-revoke-emudhra-intermediates-after-private-key-compromise.html), [eMudhra statement](https://emudhra.com/en/post/certificate-security-incident).

## The attack it enables

**Man-in-the-middle on outbound calls.** Someone on the network path shows your program a forged certificate for an API it calls. Your program accepts it, and the attacker reads or changes the traffic. You are exposed whenever *your* program is the client checking someone else's certificate.

## Timeline

| Date | Event |
|---|---|
| 2025-04-30 | Key ceremony creates the weak keys |
| 2026-10-17 | Reported. Chrome and Firefox block the intermediates the same day |
| 2026-10-18 | eMudhra revokes its 8 intermediates |
| 2026-10-19 | Sectigo revokes its 8 cross-signs. Apple blocks |
| 2026-10-20 | Microsoft blocks. Node.js fix merged |
| 2026-10-21 | Node.js releases the fix |

## Who is affected

1. **Browser users: safe now.** Browsers get block lists pushed to them (Chrome CRLSet, Firefox OneCRL) with no update needed.
2. **Everything that is not a browser: the long tail.** Language runtimes, container images, embedded devices and appliances with an old trust store stay open to man-in-the-middle until someone updates and rebuilds them. This is where CI images and backend services sit.
3. **Code signing.** Two intermediates were trusted by Microsoft for signing programs, so fake "legitimately signed" programs were possible.
4. **Cross-signs.** Sectigo had signed the same keys from its own root, `AAA Certificate Services`. That was a second valid path, so it had to be revoked too.
5. **Java.** Java ships its own root list, which includes `emSign ECC Root CA - G3` (the root that matters here), and does no revocation checking by default. The disclosure page listed the Java fix as "pending" on October 22.

What happens to the certificates and roots:

- All 16 intermediate certificates (8 from eMudhra, 8 Sectigo cross-signs) are revoked as compromised. Browsers block them.
- Mozilla requested an emergency distrust of `emSign ECC Root CA - G3` and shipped an NSS update that removes it. Python `certifi` 2026.10.19 removes it too and keeps the unaffected emSign roots. Whether eMudhra's other roots stay depends on the root programs.

Mitigating facts: eMudhra says only one of the eight was live in production and none issued customer certificates, based on its own checks and public CT logs. This is eMudhra's claim, not independently verified. No sign of exploitation was reported. The real cost is cleanup: eMudhra estimated about 30,000 certificates under `emSign ECC Root CA - G3` must move to a new root.

> [!info]- Details: Python and Java specifics
> - Python minimum versions: certifi 2026.10.19, pip 26.2.2, uv 0.13.4, sigstore 4.5.1, pixi 0.82.1. Then rebuild container images ([discuss.python.org](https://discuss.python.org/t/information-for-python-users-regarding-emudhra-emsign-ca-certificate-security-incident/109576)).
> - A CPython `ssl` denylist ([python/cpython#160040](https://github.com/python/cpython/issues/160040)) was still open on October 22.
> - Java: [JDK-8319187](https://bugs.openjdk.org/browse/JDK-8319187) "Add three eMudhra emSign roots" (emSign Root CA - G1, emSign Root CA - G2, emSign ECC Root CA - G3) landed in JDK 22 in November 2023, backported to 8u, 11u (for example [openjdk/jdk11u-dev#2290](https://github.com/openjdk/jdk11u-dev/pull/2290), 11.0.22), 17u and 21u.
> - Unverified: whether OpenJDK ships the Java fix out of band or in a quarterly update. See the Java section in [[notes/ca-certificates-and-trust-stores|CA Certificates & Trust Stores]] for the `keytool` stopgap.

## Affected chains

The eight intermediates are `EM DV TLS CA`, `EM OV TLS CA`, `EM EV TLS CA` and `EM OV CS CA`, each in two generations, G3B and G3C. They chain to:

- `emSign Root TLS CA - G3` / `emSign Root CS CA - G3` (direct issuers), cross-signed by `emSign ECC Root CA - G3`
- `AAA Certificate Services` (Sectigo cross-signs)

Which stores trust which root (from the disclosure page `roots.json`):

| Root | Mozilla | Chrome | Apple | Microsoft |
|---|---|---|---|---|
| `emSign Root TLS CA - G3`, `emSign Root CS CA - G3` | no | no | no | yes |
| `emSign ECC Root CA - G3` | yes | yes | yes | yes |
| `AAA Certificate Services` | removed | removed | yes | yes |

So for lists copied from Mozilla (Linux distros, certifi, Node), the dangerous path runs through `emSign ECC Root CA - G3`, or through `AAA Certificate Services` where an old list still has it.

> [!info]- Details: key types
> All 16 intermediates and the three emSign G3 roots are P-384 ECC. `AAA Certificate Services` is RSA 2048.

## Two ways to fix it

| Fix | How | Where it lives |
|---|---|---|
| A. Patch the trust store | Remove the roots the bad intermediates chain to | The OS / container base image |
| B. Patch the runtime | The runtime ships a built-in block list of the bad keys | The compiled program |

Fix A is blunt: other, legitimate certs also chain to those roots, so removing a root can break valid sites. Fix B is precise, but every runtime has to ship it.

Fix status at the time:

- **Go:** fixed in 1.26.10+ and 1.27.3+. Older patch releases of the same version line (for example 1.26.8) are still exposed, so the exact patch version matters.
- **Node.js:** fixed in 22.23.4, 24.22.1 and 26.12.1 (released October 21).
- **Linux distros:** patched `ca-certificates` packages, for example Debian `20250419~deb12u2` / `20250419+deb13u1`, Alpine `20261020-r0`, Chainguard `20260909-r3`. The disclosure page keeps the full table.

> [!info]- Details: how the runtime fixes work
> - Go: CVE-2026-111943, [golang/go#82345](https://github.com/golang/go/issues/82345). The block list is `distrustedSPKIHashes` in `crypto/x509/verify.go`: 8 SHA-256 hashes of the intermediates' public keys (SPKI). Because it blocks by key, it also covers the Sectigo cross-signs, which carry the same keys.
> - Node.js: the same 8 hashes, checked during certificate verification and reported as `X509_V_ERR_CERT_REVOKED` ([nodejs/node#66871](https://github.com/nodejs/node/pull/66871), merged October 20).

For old runtimes that will not get fix B, apply fix A by hand. On Debian-based images:

```bash
sed -i \
  -e 's|^mozilla/emSign_ECC_Root_CA_-_G3.crt|!&|' \
  -e 's|^mozilla/Comodo_AAA_Services_root.crt|!&|' \
  /etc/ca-certificates.conf && update-ca-certificates
```

A `!` at the start of a line in `/etc/ca-certificates.conf` marks that root as distrusted. Test your external calls afterwards, because removing a widely used root can break chains you rely on.

## Testing whether an image is safe

Checking root names or package versions is not enough, because cross-signs give the same intermediate several paths. The reliable test is to try the real attack chains. The disclosure page publishes `chains.zip`: the 12 chains its test hosts serve (3 TLS kinds x 2 generations x 2 roots).

Results at the time (how many of the 12 chains each image still trusted):

| Image | Trusted |
|---|---|
| `gcr.io/distroless/static-debian13`, `base-debian13`, `static:nonroot` | 0 |
| `gcr.io/distroless/static-debian12` | 12 |
| `cgr.dev/chainguard/static` | 0 |
| `golang:1.26.10` / `-trixie` / `-bookworm`, `golang:1.27.3` | 0 |
| `golang:1.26.10-alpine` | 6 |

Two lessons. The `debian12` and `debian13` variants of the same distroless image differ completely, so pick the variant on purpose. And a version number alone tells you little: distroless debian13 had an older `ca-certificates` (`20250419`) but was clean, because that package never contained the two affected roots.

The runtime fix works on its own, regardless of the trust store. A small Go program with the unpatched debian12 trust store accepted all 12 chains on Go 1.26.8 and rejected all 12 on Go 1.26.10.

> [!info]- Details: test commands
> Check one chain against a trust store. `OK` means the store still trusts a compromised intermediate:
>
> ```bash
> openssl verify -CAfile /path/to/ca-certificates.crt -untrusted chain.pem chain.pem
> ```
>
> Pull the trust store out of a container image without running it:
>
> ```bash
> crane export --platform linux/amd64 IMAGE - | tar -xO etc/ssl/certs/ca-certificates.crt > bundle.pem
> ```
>
> The Go test used `x509.Certificate.Verify`.

## Why a fix takes a rebuild

A container image is a frozen snapshot:

```
compile     program built with the Go version in go.mod   <- runtime fix (B) lives here
layer       program placed on top of a base image         <- trust store fix (A) lives here
deploy      the cluster runs that exact image
```

Running containers keep the old program and the old trust store until the image is rebuilt and redeployed. Base images pinned by digest do not move on their own.

> [!info]- Details: build pitfalls
> - Build systems that pass the Go version as a build argument (for example `--build-arg GOVERSION=...`) need that value bumped too.
> - Official `golang` images set `GOTOOLCHAIN=local`, so a newer `go` line in `go.mod` fails the build instead of downloading the newer Go.

## Takeaways

- Small market share does not limit damage. Every trusted CA can sign for every domain.
- Weak key generation is invisible to normal certificate checks. Only blocking or removing the root helps.
- Revoke every cross-signed copy of a bad key, not only one path.
- Anything without a push channel (Go, Python, Node, Java, distro packages) needs a release, then a rebuild and redeploy.
- Test images against the real attack chains, not root names or version numbers.
