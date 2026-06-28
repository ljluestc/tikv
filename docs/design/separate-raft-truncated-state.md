# Separate `RaftTruncatedState` from `RaftApplyState`

- Tracking issue: https://github.com/tikv/tikv/issues/3737
- Status: Draft / proposal

## Summary

`RaftTruncatedState` is currently embedded as a field of `RaftApplyState`
(both defined in `kvproto`'s `raft_serverpb`). This document proposes
decoupling the two concepts so that raft-log compaction state is no longer
carried inside the state-machine apply state.

## Motivation

`RaftApplyState` and `RaftTruncatedState` describe two distinct concerns:

- `RaftApplyState` belongs to the state machine (RocksDB) that applies raft
  logs: `applied_index`, `commit_index`, `commit_term`.
- `RaftTruncatedState` belongs to raft-log compaction: `index`, `term` of the
  last truncated entry.

Coupling them has two downsides:

1. **Conceptual coupling.** Log-compaction bookkeeping is persisted and updated
   on the apply path even though it is logically independent of applying user
   writes.
2. **Write amplification.** Every apply-state write carries the truncated state.
   A smaller, dedicated `RaftApplyState` reduces the size of the value written
   on the hot apply path and may improve performance.

## Current state (as of this proposal)

- `RaftApplyState.truncated_state` is read/written through generated accessors
  (`get_truncated_state` / `mut_truncated_state` / `set_truncated_state`)
  in roughly 40+ sites across `components/raftstore` (apply FSM, peer FSM,
  `peer_storage`, snapshot apply, region meta, and tests).
- The apply state — including the embedded truncated state — is persisted under
  the apply-state key and restored on restart and snapshot apply
  (`peer_storage.rs`).
- `components/raftstore/src/store/region_meta.rs` already exposes a *separate*
  `RaftTruncatedState` view for the debug/region-meta read path, which is a
  natural model for the in-memory split.

## Proposed approach

The change spans `kvproto` and TiKV and must be backward compatible, so it is
staged:

1. **kvproto**: add a standalone `RaftTruncatedState` persisted value (new key)
   while keeping the embedded field for compatibility. No TiKV behavior change.
2. **TiKV write path**: write the truncated state to the new dedicated key in
   addition to the embedded field (dual-write), so old and new readers both
   work during rollout.
3. **TiKV read path**: prefer the dedicated key, falling back to the embedded
   field when the dedicated key is absent (upgrade from an older cluster).
4. **Encapsulation**: route all access through accessors on `PeerStorage`
   (and the apply delegate) instead of touching `apply_state.mut_truncated_state()`
   directly, to localize the eventual removal of the embedded field.
5. **Cleanup (later major release)**: once all supported upgrade paths populate
   the dedicated key, stop writing the embedded field and shrink
   `RaftApplyState`.

## Compatibility

- Persistent data format changes: yes. Requires dual-write + fallback-read
  during at least one release window before the embedded field can be dropped.
- Snapshot apply, online/offline downgrade, and `tikv-ctl` inspection paths
  must all understand both layouts during the transition.

## Testing plan

- Unit tests in `peer_storage` for dual-write / fallback-read.
- Upgrade tests: cluster started on the old layout, restarted on the new
  binary, verifying truncated state is preserved and compaction still works.
- Snapshot send/apply across mixed-version peers.

## Out of scope for the first PR

The first PR introduces this design note and the access-encapsulation seam in
TiKV. The `kvproto` schema change, dual-write/fallback-read, and the eventual
field removal are follow-ups tracked under the same issue.
