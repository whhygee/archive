---
title: "Git proactiveAuth - default 401s"
---

## Problem

`go get` / `go mod download` triggers Git's HTTP challenge-response: an unauthenticated request → 401 → retry with credentials. At scale, the 401s trigger GitHub's anti-abuse system and block CI runner NAT IPs.

## Fix: proactiveAuth

`git config http.https://github.com/your-org/.proactiveAuth auto` tells Git to send credentials on the first request, skipping the 401.

## Why a credential helper is needed

`actions/checkout` authenticates via `.extraheader` (an `AUTHORIZATION` header injected into git config), not via Git's credential helper. proactiveAuth only triggers the **credential helper** path. Without a credential helper configured, `git fetch` after checkout fails with `could not read Username`.

The wrapper sets up a global credential helper using the same token:

```yaml
git config --global credential.https://github.com.helper \
  '!f() { echo "username=x-access-token"; echo "password=${TOKEN}"; }; f'
```

## Why proactiveAuth must be disabled during checkout

`actions/checkout` injects its own auth headers via `.extraheader`. If proactiveAuth is also enabled, Git sends credentials via both the credential helper AND the extraheader simultaneously, which conflicts and breaks checkout. The wrapper disables proactiveAuth before checkout and re-enables it after.

## persist-credentials interaction

- `true` (default): credentials stay in local git config after checkout. The wrapper sets up the credential helper and enables proactiveAuth.
- `false`: user explicitly opts out of persisted credentials. The wrapper skips credential helper setup to respect that intent. proactiveAuth is also skipped since it would fail without credentials.

## Git version string bug

The runner Dockerfile sets `ENV GIT_VERSION=v2.53.0`. Git's `GIT-VERSION-GEN` script checks `if test -z "$GIT_VERSION"` — since the env var is already set, it skips version detection (including `v` prefix stripping) and uses `v2.53.0` as-is. This produces `git version v2.53.0` instead of `git version 2.53.0`. Go's version parser regex (`git version\s+(\d+\.\d+(?:\.\d+)?)`) expects digits after `git version` and fails on the `v`. Fix: pass `GIT_VERSION=${GIT_VERSION#v}` to `make`.
