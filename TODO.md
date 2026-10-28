# TODO before submitting upstream

Review notes on this PR (oOraph/dragonfly#2, fixing oOraph/dragonfly#1). Delete this file before
opening the upstream PR.

No correctness bug found: FIFO ordering holds (appending never preempts, so OnChange baseline still
precedes its journal entry in the same blob), the full-sync cut and the EOF token go out only after
the queue drains, and every path that starts `flush_fb_` stops it.

## Resolved (2026-10-28)

1. **Default `0` semantics.** Egress is now recorded by the flush fiber just before the write, as
   the producers did. The post-flush sleep stays traversal-only on purpose and the commit message
   says so (a mutation fiber's sleep lands on every command queued behind it).
2. **Memory bound.** Flag help, docs and commit message now say: per shard *and per running
   snapshot*, soft by one chunk per blocked producer. Verified in `EngineShard::UsedMemory`: the
   queued strings live on the thread's default mimalloc heap, not the data heap, so they count
   toward RSS and `snapshot_serialization_bytes` but not `used_memory`.
3. **Direct-write branch after `StopFlushFb`.** Unreachable: `UnregisterOnChange` waits on
   `change_cb_latch_` (held across `OnChange`) and the journal's `UnregisterOnChange` takes
   `cb_mu_` (held across `CallOnChange`), so no producer is in flight when the fiber is joined.
   Now `LOG(DFATAL)` with the direct write as the release fallback.
4. **Tests.** `RdbTest.SnapshotOutputBuffer` saves while another fiber overwrites not yet
   traversed keys. `test_replication_full_sync_output_buffer_replica_stalls` (`0` / `1mb`) pauses
   the replica mid sync, asserts `snapshot_serialization_bytes` above the limit, lets
   `replication_timeout` drop the replica, checks writers release and the queue drains, then
   resumes the replica and compares the data.
5. `pending_bytes_` counts string capacity.
6. Flush fiber priority: one comment at the fiber creation (NORMAL even in background mode).
7. Commit message now cites the PR body's numbers (production 16384 x 1.5 MB, synthetic repro).

## Remaining

8. PR body links `oOraph/dragonfly#1`; upstream it must say `Fixes dragonflydb/dragonfly#<new>`.

## Upstream plan

1. Create the upstream issue from oOraph/dragonfly#1's text.
   - Decide whether the repro's member names (`org%d/repo%d/...`) and the "cache index"
     description are fine to publish as is.
2. Update commit message and PR body to link the new issue; delete this file; push
   `HEAD:refs/heads/full-sync-async-push` to `origin`.
3. Open a **draft** PR on `dragonflydb/dragonfly` from `oOraph:full-sync-async-push`.
   - Upstream may ask to sign the CLA (`CLA.txt`).
