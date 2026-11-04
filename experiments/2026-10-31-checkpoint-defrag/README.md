# Checkpoint defragmentation, rerun (2026-10-31)

Base deployment commit: `a5a29cb6` (site profile `config/site-tp4-ring.json`,
image `spark-ds41f:1030`, baseline `2026-10-30-spark-ds41f-1030`). Owner
authorized the rerun after the 1030-TP4 deployment. The previous attempt
(`experiments/2026-10-02-weight-defragmentation`) paused because its 1 GiB
cgroup OOMed mid-run on every node; serving was stopped first this time.

## Inputs

- Four nodes dgx1–dgx4, one 48-shard checkpoint per node (476 GiB), snapshot
  hardlinked under `~/.cache/huggingface/hub` and `~/models/`.
- Fragmentation before (shards with current extents above best):
  dgx1 45/48 (worst 153/4), dgx2 48/48 (worst 1457/48),
  dgx3 45/48 (worst 532/4), dgx4 39/48 (worst 189/48).

## Method

Same per-shard protocol as the 10-02 run ([defragment.py](defragment.py)):
checksum, `e4defrag -v`, checksum, abort below 4 GiB MemAvailable. Two
changes: the memory limit is 8 GiB instead of 1 GiB (the observed failure
cause), and the pre-checkpoint hash is recorded rather than compared to a
content-addressed blob name, because this site's snapshot is hardlinked under
`~/models/` and is not hash-named; before/after equality still guards each
shard, and per-node hashes are cross-checked after the run.

Run per node in a bounded transient unit, all four in parallel, with serving
stopped inside a coordinated window (lab window + hold file), one causal
variable against 10-02 (memory limit) plus the serving stop the 10-02
conclusion required:

```
systemd-run --unit=spark-defrag --collect --quiet \
  -p MemoryMax=8G -p MemorySwapMax=0 -p Nice=19 \
  -p IOSchedulingClass=idle -p IOWeight=1 -p CPUQuota=100% \
  python3 /tmp/defragment.py --root <snapshot> --output /tmp/defrag-run
```

Canary (dgx1, shard 3, 7.4 GiB, extents 5/4): unit finished in 7.7 s, exit 0,
hashes matched, extents unchanged — `e4defrag` reports success without moving
extents when the gain is marginal.

## Runs

Full run 2026-10-31 08:20–08:31 CST (about 10 minutes wall, all four nodes in
parallel; MemAvailable never fell below 117.4 GiB on any node; no OOM, no
abort, no low-memory termination):

| node | shards processed | skipped already ideal | per-shard defrag time | improved | unchanged | OOM/abort |
|---|---|---|---|---|---|---|
| dgx1 | 45 | 3 | 521 s total, max 99.4 s | 2 | 43 | none |
| dgx2 | 48 | 0 | 640 s total, max 124.8 s | 0 | 48 | none |
| dgx3 | 45 | 3 | 620 s total, max 121.9 s | 5 | 40 | none |
| dgx4 | 39 | 9 | 573 s total, max 122.1 s | 1 | 38 | none |

Largest single moves: dgx3 shard 38 503→103 and shard 39 420→22; dgx1 shard 4
60→5. The most fragmented files (dgx2 shards 47/48 at 1457/1453 extents over
101 GiB each) did not move at all, and their 7 GiB siblings likewise: with
the filesystem 84–120 GiB-shard-size fragmented, `e4defrag` finds no large
enough free run to relocate to, so it succeeds without improving.

Data integrity: every shard's SHA-256 before equals after (177/177 processed
records `verified: true`); the 48 per-shard hashes agree across all four
nodes; no inode/size/mtime surprise. Total extents across the fleet moved
23,169 → 22,140 (−4.4%). Non-ideal shard counts are essentially unchanged
(177 → 176): the corrections that ran were small-locality moves.

Per-node machine records (before/after extent reports and per-shard
checksums): [runs/](runs/).

## Conclusion

The 8 GiB limit fixed the 10-02 failure: the full 48-shard pass completed on
every node in about ten minutes with serving stopped, data verified intact.
But `e4defrag` only corrected shards with a nearby large free run; the
heavily fragmented 7 GiB and 101 GiB shards — exactly the ones the 10-02 run
hoped to fix — did not move, because the block group free space has no
extent large enough to absorb them. Extent counts dropped only 4.4% fleet
wide and the non-ideal count barely budged, so no measurable boot-time
benefit is expected; the 10-02 note that loading is not established as
fragmentation-bound stands. This closes the defrag lever: further improvement
would need a rewrite copy (read-and-rewrite the file, e.g. copy through a new
file and rename), which is a different operation than `e4defrag` and was not
attempted here.
