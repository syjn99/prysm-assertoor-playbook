# prysm-assertoor-playbook

[Assertoor](https://github.com/ethpandaops/assertoor) playbooks that reproduce
Prysm's Go end-to-end **evaluators** as declarative tests against a running
network (e.g. one spun up with `ethereum-package`).

## [`checks/`](./checks) — standalone evaluator replacements

| Playbook | Replaces (Prysm evaluator) | Notes |
|---|---|---|
| [fork-transition.yaml](checks/fork-transition.yaml) | `AltairForkTransition` … `FuluForkTransition` (#22–27) | `check_consensus_slot_range` to the fork epoch, then `check_consensus_api` on `/eth/v2/beacon/blocks/head` requiring `version == <fork>`. |
| [prysm-metrics.yaml](checks/prysm-metrics.yaml) | `MetricsCheck` (#5) | Memory threshold via `check_http_metrics`; cache- and p2p-ratio checks via `run_javascript`. |
| [prysm-peers.yaml](checks/prysm-peers.yaml) | `PeersCheck` (#6) | `run_shell` + `grpcurl` against Prysm's `ethereum.eth.v1alpha1.Debug/ListPeers`; fails on negative peer scores. |
| [optimistic-sync.yaml](checks/optimistic-sync.yaml) | `OptimisticSyncEnabled` (#31) | `check_consensus_sync_status` with `expectOptimistic`, around a shell-driven Engine API `SYNCING` fault toggle. |
| [validator-sync-participation.yaml](checks/validator-sync-participation.yaml) | `ValidatorSyncParticipation` (#9) | `run_javascript` scans recent blocks and counts `sync_aggregate.sync_committee_bits` (95% normal, 90% in the Altair fork epoch). |
| [blob-limits.yaml](checks/blob-limits.yaml) | `BlobsIncludedInBlocks` (#28), `BlobLimitsRespected` (#29) | `check_consensus_block_proposals` with `minBlobCount`; BPO limit enforcement stays partially Prysm-specific. |
| [fee-recipient.yaml](checks/fee-recipient.yaml) | `FeeRecipientIsPresent` (#10) | Inspects proposed blocks for the expected fee-recipient; key→fee-recipient mapping is config-specific. |

## [`lifecycle/`](./lifecycle) — validator lifecycle shards

Smaller shards of `ethereum-package`'s large `run_lifecycle_test` (deposits,
BLS changes, exits, slashings, finality recovery, cleanup). See
[`lifecycle/README.md`](./lifecycle/README.md) for the phase-by-phase mapping,
caveats, and index-range guidance.
