---
title: Migrating Small-File NFS Volumes (Filestore HDD to SSD)
---

Record of migrating a CD platform's file storage from GCP Filestore Basic HDD to Enterprise SSD multishare. Two environments: a lab volume with 351K files / 1.53 GB (avg ~4.4 KB per file) and a prod volume with 431K files / 49.9 GB (avg ~116 KB per file). Both sources were 100 GiB Basic HDD instances (spec ~600 IOPS). What follows is what we tried, what it measured, and what failed.

## Why small files were slow

Every file operation on NFS is a remote procedure call (RPC), a network round trip to the server. Copying one file takes at least three: open, read, close. Round trips inside the region ran ~1ms, so a 4 KB file cost ~3ms of waiting for microseconds of data transfer.

File size, not total bytes, determined our copy speed:

- Lab (avg 4.4 KB/file): RPC-bound. ~50 MB/min at 16 parallel workers.
- Prod (avg 116 KB/file): throughput-bound. ~940 MB/min with the same tooling, same parallelism, same source tier.

We initially estimated prod at ~17 hours by extrapolating the lab MB/min figure. Actual prod pre-warm took 53 minutes. The 19x error came entirely from ignoring the file-size difference.

## Everything we tried, with numbers

All against the lab source (100 GiB Basic HDD, ~600 IOPS spec):

| Attempt | Measured throughput | Outcome |
|---|---|---|
| Serial `cp` | ~7-10 MB/min | Too slow. One RPC in flight at a time |
| `tar -cf - . \| tar -xf - -C /dest` streaming pipe | ~7.5 MB/min (1.7 GB in 3h46m, killed) | No better than serial cp. tar opens each file one at a time, single-threaded, same RPC cost |
| Single rsync | Never finished, killed | Same class as serial cp |
| `xargs -P 16` parallel cp | ~50 MB/min | 5-7x over serial. First run failed its own verify because the source changed mid-copy (app was live) |
| `xargs -P 16` parallel rsync | ~46-50 MB/min | Same as parallel cp, plus idempotent re-runs |
| `xargs -P 64` parallel rsync | ~54 MB/min | +8% over P=16. Concurrency curve flat past ~16 workers |
| 3 pods × 16 workers each (48 total) | ~59 MB/min aggregate | +18% for 3x the pods. All 3 pods landed on one node and the CSI driver deduplicated the mount, so they shared one NFS client. Even accounting for that, not a lever |
| Parallel delete, `xargs -P 32 rm -rf` | 15x over serial (10h → 39min for 358K entries) | Same physics as copy: unlink is one RPC per file |

Past ~16 workers the bottleneck was the server-side IOPS ceiling (the maximum I/O operations per second the NFS server handles, which on Filestore scales with tier and capacity). No client-side change moved it after that.

### Approaches we evaluated and rejected without full runs

- **Zip first, transfer one big file, unzip.** Rejected on analysis: creating the archive costs one open+read per source file, which is the entire expensive part, and then unpacking costs one open+write per file on the destination. Strictly worse than the streaming tar pipe we did measure, which was itself no better than serial cp. Archive-first wins only when reads are cheap and transport is expensive (internet upload, tape). Ours was the opposite: fast network, expensive reads.
- **Staging through GCS.** Rejected on analysis: `gsutil -m`'s parallel uploaders provide the same concurrency `xargs -P` already provided, and the source-side per-file read cost is unavoidable. Transport was never the problem.
- **`nconnect` mount option** (multiple TCP connections per mount). Rejected via documentation: GCP supports it on Zonal, Regional, Enterprise, and High-scale tiers. Basic HDD, our source, is not on the list.
- **In-place tier upgrade of the source.** Rejected via documentation: Filestore tier is immutable on an existing instance. The only path to a faster tier is create-new-and-copy.
- **Snapshot/backup restore to the new tier.** Rejected via documentation: Filestore backups cannot be restored onto an Enterprise multishare instance.

## What shipped: pre-warm + delta with parallel rsync

1. **Pre-warm with the application live.** Parallel rsync copied the bulk. Lab: 55 min. Prod: 53 min. Zero downtime.
2. **Cutover.** Scaled the application to zero (source frozen), re-ran the identical rsync job. The delta pass stat-walks every file but transfers only the drift. Lab delta: 21 min. Prod delta: 25.5 min.
3. Swapped the volume references in the StatefulSet, scaled back up.
4. **Verified after the app was back up**, not inside the downtime window.

rsync is idempotent: a re-run skips files whose size and modification time match, so each pass after the first costs a stat walk (checking every file's metadata) plus only the changed bytes. `cp` re-copies everything, which is why the first parallel-cp attempt was abandoned for rsync.

The job that ran in both environments:

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

- `--inplace --whole-file --no-compress` because the temp-file rename, the delta algorithm, and compression all cost CPU and bought nothing on a fast local network where destination files were either absent or fully replaced.
- The trailing slash on `"$1/"` makes rsync sync directory *contents* into the matching destination directory.

### The exit-23 failure and fix

First pre-warm attempt (no exit-code wrapper) died at 38 minutes: the application's retention job deleted directories that `find` had already enumerated, rsync exited 23 ("some files were not transferred"), and under `xargs` one nonzero worker failed the whole job.

Exit 23 and 24 ("files vanished during transfer") are correct behavior against a live source: a file deleted from the source does not need to reach the destination. The wrapper above accepts 23/24 per worker and propagates everything else. With it, pre-warm runs survived multiple vanished-directory events in both environments and exited 0.

## Verification: what the data showed

- **`du -sb` disagreed with reality by 2x on one instance.** The lab HDD source reported 2.98 GB via `du -sb`; the true content (sum of `stat().st_size`) was 1.53 GB. The prod HDD source reported accurately. Same tier, different accounting. We chased a phantom "48% data loss" for a day because of this. The measurement that matched on every volume:

  ```sh
  find /vol -type f -printf '%s\n' | awk '{s+=$1} END {print s}'
  ```

  BusyBox `find` (Alpine default) lacks `-printf`; we installed `findutils` and called `/usr/bin/find`.

- **Destination measured slightly larger than source after every delta pass**: dst/src ratios of 1.0105, 1.0257, 1.0336, 1.0382 across four verify runs. Cause: the destination retained files that source-side retention had deleted after the copy. Below 0.99 would have meant real missing data; we never saw it.

- **Verifying against a live source produced garbage.** One verify ran after the application had resumed: destination showed 4,774 more files than source purely because both sides had drifted apart. Totals are only comparable while the source is frozen.

- **The verify pass cost ~2 hours** for ~430K files on both sides (it is a full metadata walk, same cost class as the copy's stat walk). Our first cutover had verification embedded in the copy job, which put those 2 hours inside the downtime window. Lab downtime was 2h47m as a result, of which rsync was 22 minutes. We split verify into a separate job that runs after the app is back up, with both volumes mounted read-only. Prod downtime was 38 minutes.

## Monitoring: what worked and what backfired

- `df` answered instantly throughout: the kernel serves cached filesystem statistics (`statfs`), issuing no RPCs. This was the progress signal for every run.
- `du -sh` and `find | wc -l` against the volume under copy load took 3+ minutes or hung outright: they are full metadata walks, one RPC per file, competing with the copy for the same server IOPS. We made this mistake three separate times before it stuck.
- `df` wraps long NFS device names onto a second output line, which broke our first monitoring script's `awk`. `df -P` (POSIX format) does not wrap.

## Failures and surprises during cutover

- **GitOps auto-apply reverted a manual scale-down.** The deploy workflow re-applies manifests on every merge to main, and the manifest declared `replicas: 2`. An unrelated merge scaled the application back up in the middle of a copy, contaminating the destination. The fix used at cutover: merge the replica change through the pipeline rather than scaling by hand.
- **Fresh PVCs against a multishare instance bound serially.** At prod cutover, three destination volumes had never been mounted (WaitForFirstConsumer binding). The CSI driver's share-creation calls against the same multishare instance were serialized, each retrying on `Aborted: all eligible filestore instances are busy`. Binding took ~11 minutes, all inside the downtime window. The volume that had been pre-mounted by the copy job bound instantly. Mounting each destination volume once from a throwaway pod before the window would have removed those 11 minutes.
- **Deleting the old PVCs destroyed the backends, by design.** Reclaim policy was `Delete`, so PVC deletion destroyed the Filestore instances. We removed the PVC definitions from the manifests first; deleting the objects while definitions remained would have had the GitOps loop recreate them as empty volumes.
- **Multi-hour `kubectl wait` commands died on credential expiry.** The cloud auth plugin's tokens expire around the 1-hour mark and cannot re-prompt inside a non-interactive command. Every long blocking wait eventually failed this way; short polling loops did not.
- **Everything ran on lab first.** The exit-23 failure, the du-vs-stat discrepancy, the verify-inside-copy-job downtime mistake, and the GitOps race were all discovered on lab at zero production cost. The prod cutover hit exactly one new problem (the serial PVC binding above).

## The cutover window, measured

```
delta time ≈ (file count × per-file stat cost at your concurrency) + (changed bytes / throughput)
```

At 16 workers against Basic HDD, the stat check cost ~3.6ms per file. Prod's 431K files predicted ~26 minutes of stat walk; the actual delta rsync ran 25.5 minutes. The stat walk dominated and its cost was independent of how much had drifted, which made the window the same whether pre-warm had finished an hour or three days earlier.

Prod cutover, wall-clock breakdown: drain 1s, delta rsync 25.5 min, deploy pipeline ~30s, PVC binding ~11 min, application startup ~2 min. Total ~38 minutes.
