---
title: Migrating Small-File NFS Volumes (Filestore HDD to SSD)
---

Lessons from migrating a CD platform's file storage from GCP Filestore Basic HDD to Enterprise SSD multishare. The data was hundreds of thousands of small files on NFS. Everything here applies to any NFS-to-NFS bulk copy where the volume is full of small files.

## Why small files are slow on NFS

Every file operation on NFS is a remote procedure call (RPC), a network round trip to the server. Copying a single file takes at least three: open, read, close. Inside a GCP region each round trip is about 1ms, so a 4 KB file costs ~3ms of waiting for microseconds of actual data transfer.

This means **how big your files are matters more than how much data you have**:

- 350K files averaging 4 KB (1.5 GB total): the copy is RPC-bound. Nearly all time goes to round trips; the disk is mostly idle.
- 430K files averaging 116 KB (50 GB total): each RPC carries 26x more data, so the copy is throughput-bound instead, running close to the disk's limit.

We measured both against the same source tier. The small-file volume copied at ~50 MB/min. The medium-file volume copied at ~940 MB/min. Same tooling, same parallelism, same NFS tier. The only difference was file size.

So when estimating a migration, get the file count and the average file size, not just the total bytes. Extrapolating speed from a volume with a different file-size mix can be wrong by 20x. It was for us.

## What actually speeds up the copy

Measured against a single 100 GiB Basic HDD source (spec ~600 IOPS):

| Approach | Throughput | Verdict |
|---|---|---|
| Serial `cp`, `tar` pipe, or single rsync | ~7-10 MB/min | Baseline: one request in flight at a time |
| `xargs -P 16` parallel rsync | ~50 MB/min | The big win, 5-7x over serial |
| `xargs -P 64` | ~54 MB/min | Only +8% over P=16. We hit the wall |
| 3 pods x 16 workers each | ~59 MB/min | +18% for 3x the pods. Not worth it |
| Parallel delete (`xargs -P 32 rm -rf`) | 15x over serial | Deleting has the same physics as copying |

The lesson: parallelism helps enormously at first, then stops. Around 16 concurrent workers the bottleneck shifts from the client to the server-side IOPS ceiling (the maximum I/O operations per second the NFS server will do, which on Filestore scales with tier and provisioned capacity). Once you are IOPS-bound on the server, nothing you do on the client moves the number.

### Ideas that sound good but don't work

- **"Zip everything first, then copy one big file."** Creating the archive still has to open and read every source file, which is exactly the expensive part. Then you pay again to unpack on the destination. Archiving-first only wins when reading is cheap and transport is expensive (uploading over the internet, writing to tape). Copying between two NFS volumes in the same region is the opposite situation: the network is fast, reading the files is what's slow.
- **Staging through object storage (GCS etc.).** Same trap. The parallel uploaders in `gsutil -m` give you the same concurrency you already had with `xargs -P`. You cannot avoid reading every file from the source.
- **Streaming tar** (`tar -cf - . | tar -xf - -C /dest`). Feels like it should turn thousands of small reads into one smooth stream. On NFS it doesn't: tar still opens and reads each file one at a time, single-threaded. We measured it. Identical to serial cp.
- **The `nconnect` mount option.** Opens multiple TCP connections per mount and genuinely helps, but only on tiers where Google supports it (Zonal, Regional, Enterprise, High-scale). Basic HDD is not on the list.
- **Spreading across multiple pods.** In theory each pod brings its own NFS client and its own request pipeline. Two problems in practice: pods that land on the same node share one mount (the CSI driver deduplicates it; use podAntiAffinity if you attempt this), and even with separate mounts the server-side limit still binds. We got +18% from 3x the pods.
- **Upgrading the source tier in place.** Filestore tier is immutable once an instance exists, like its IP address. The only way to a faster tier is to create a new instance and copy into it, which is the problem you were trying to avoid.
- **Restoring a snapshot/backup onto the new tier.** Filestore backups cannot be restored to an Enterprise multishare instance. Dead end for HDD-to-multishare specifically.

## The pattern that works: pre-warm, then delta

Once you accept the throughput ceiling, stop optimizing the copy and minimize *downtime* instead:

1. **Pre-warm** while the application is still running: parallel rsync copies the bulk of the data. Takes as long as it takes; nobody is waiting.
2. **Cutover**: stop the application so the source is frozen, then run the same rsync job again. This delta pass checks every file but only transfers what changed since the pre-warm. Downtime is bounded by the stat walk plus a small transfer, not by the full copy.
3. Swap the volume references and restart the application.
4. **Verify after the app is back up**, not during the downtime window.

rsync makes this possible because it is idempotent: re-running it skips files whose size and modification time already match, so each pass after the first costs a stat walk (checking every file's metadata) plus only the changed bytes. `cp` has no such concept; every re-run copies everything from zero.

The parallel-per-directory pattern we settled on:

```sh
find . -maxdepth 1 -mindepth 1 -type d -print0 \
  | xargs -0 -r -P 16 -I {} sh -c '
      rsync -a --inplace --whole-file --no-compress "$1/" "/dest/$1/"
      rc=$?
      if [ $rc -eq 0 ] || [ $rc -eq 23 ] || [ $rc -eq 24 ]; then exit 0; fi
      echo "WORKER FAIL $1 rc=$rc" >&2
      exit $rc
    ' _ {}
```

- `--inplace --whole-file --no-compress`: skip the temp-file rename dance, skip the delta algorithm, skip compression. All three just burn CPU when the network is fast and the destination file either doesn't exist or gets fully replaced.
- Trailing slashes matter in rsync: `src/` means "sync the contents of src into the destination directory".

### Treat rsync exit codes 23 and 24 as success during pre-warm

If the application deletes files while rsync is walking the tree (retention cleanup, log rotation), rsync reports exit 23 ("some files were not transferred") or 24 ("files vanished during transfer"). Under `xargs`, one worker exiting nonzero fails the entire job.

During a live-source pre-warm these exits are expected and harmless: a file that no longer exists on the source doesn't need to reach the destination. The wrapper above accepts 23/24 per worker and still fails on anything else. Our first pre-warm attempt did not have this and died 38 minutes in, when the app's retention job deleted directories that `find` had already listed.

## Verifying the copy without fooling yourself

- **Don't trust `du` across different NFS servers.** On one Basic HDD instance, `du -sb` reported roughly double the real content size (it counts allocated blocks, and the server's accounting differed); on another instance of the same tier it was accurate. The only measurement that means the same thing everywhere is summing actual file sizes:

  ```sh
  find /vol -type f -printf '%s\n' | awk '{s+=$1} END {print s}'
  ```

  Note BusyBox `find` (Alpine default) lacks `-printf`; install `findutils` and use `/usr/bin/find`.

- **Expect the destination to be slightly *larger* than the source.** After the delta pass, the destination still holds files the source's retention already deleted. A destination/source ratio between 1.00 and 1.05 is healthy. Below 0.99 means data is actually missing.

- **Verify while the source is frozen, or don't bother.** The comparison only means something when neither side is changing. Once the application resumes, both sides drift apart and the totals stop being comparable. If you must check later, spot-check individual files instead.

- **Verification is expensive.** It is a full metadata walk of every file on both sides, which costs about as much as the copy's own stat walk (about two hours for ~430K files each side, in our case). Keep it out of the cutover window: run it as a separate job after the application is back up, with both volumes mounted read-only. We initially built verification into the copy job and it doubled our downtime for nothing; a clean rsync exit was already the signal to proceed.

## Watching progress without slowing the copy

- `df` is free: the kernel answers from cached filesystem statistics (`statfs`), no RPCs issued. Use it to watch the destination fill up.
- `du` and `find | wc -l` are full metadata walks, one RPC per file, competing with the copy for the same limited server IOPS. Against a volume under heavy copy load they can take many minutes or appear to hang. We burned time on this more than once.
- `df` wraps long NFS device names onto a second line, which breaks naive parsing. Use `df -P` in scripts.

## Operational gotchas

- **GitOps auto-apply undoes manual scale-downs.** If a workflow applies manifests on every merge and the manifest says `replicas: 2`, your manual scale-to-zero is reverted by the next unrelated merge. Merge a `replicas: 0` change before the cutover, or accept the race.
- **Fresh PVCs against a multishare instance bind one at a time.** We assumed the new volumes would provision instantly at cutover. Instead the CSI driver's share-creation calls were serialized (each retrying on "all eligible filestore instances are busy"), and three fresh PVCs took ~11 minutes to bind, all of it downtime. Pre-provision destination volumes before the window by mounting them once from a throwaway pod.
- **Reclaim policy `Delete` destroys the backend when the PVC goes.** Check `persistentVolumeReclaimPolicy` before cleanup. And remove the PVC definitions from your manifests *before* deleting the objects, or the GitOps loop recreates them as empty volumes.
- **Long `kubectl wait` commands die when cloud credentials expire.** Auth plugin tokens have ~1h lifetimes and can't re-prompt inside a non-interactive wait. Poll in short iterations instead of one long blocking wait.
- **Rehearse everything on a lower environment.** Every technique here (parallel rsync, exit-code handling, delta timing, the verify method) was proven on our lab environment before prod. The rehearsal is also what exposed the verify-inside-the-copy-job mistake, at zero production cost.

## Estimating the cutover window

For the delta pass against a frozen source:

```
delta time ≈ (file count × per-file stat cost at your concurrency) + (changed bytes / throughput)
```

At 16 workers against Basic HDD we measured ~3.6ms per file for the stat check. For 430K files that's ~26 minutes of stat walk, plus a few minutes transferring the drift. The stat walk dominates, and its cost is fixed regardless of how much actually changed. That makes the window predictable: whether you pre-warmed an hour ago or three days ago, the cutover costs about the same.

Budget on top of that: application drain time, volume provisioning (see gotcha above), application startup, and your deploy pipeline's latency. Ours came to ~38 minutes total, 25 of which was rsync.
