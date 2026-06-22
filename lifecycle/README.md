# Validator Lifecycle Shards

`ethereum-package` has a convenient `assertoor_params.run_lifecycle_test`
switch, but the bundled test is intentionally large: the package documents it
as about 48 hours, requiring exactly 500 active validator keys, and causing
temporary unfinality. These playbooks split that test into smaller shards that
mirror its individual pieces.

Source audited:

- `ethpandaops/ethereum-package` `main` at `02675ab`
- `static_files/assertoor-config/tests/validator-lifecycle-test.yaml`
- `ethpandaops/assertoor` `master` at `2231b3e`

## Shards

| Phase | Upstream behavior | Shard | Prysm evaluators replaced |
|---|---|---|---|
| Health | Wait for at least one client to be ready. | [health.yaml](health.yaml) | `HealthzCheck` (#2) |
| Deposits | Generate 300 deposits from the lifecycle mnemonic, wait for receipt, best-effort check each validator client pair proposes a block with deposits, then wait for the last deposited pubkey to become active. | [deposits.yaml](deposits.yaml) | `ProcessesDepositsInBlocks` (#12), `ActivatesDepositedValidators` (#13), `DepositedValidatorsAreActive` (#14) |
| Unfinality | After the 300 deposits activate, assert the chain has at least 5 unfinalized epochs. | [unfinality-check.yaml](unfinality-check.yaml) | — (lifecycle precondition) |
| BLS changes | Generate 50 BLS-to-execution changes and check each validator client pair proposes a block containing at least one. Upstream runs this once during unfinality and once again after finality returns. | [bls-changes.yaml](bls-changes.yaml) | `SubmitWithdrawal` (#18), `ValidatorsHaveWithdrawn` (#19) |
| Voluntary exits | Generate 50 voluntary exits and check each validator client pair proposes a block containing at least one. Upstream runs this once during unfinality and once again after finality returns. | [voluntary-exits.yaml](voluntary-exits.yaml) | `ProposeVoluntaryExit` (#16), `ValidatorsHaveExited` (#17) |
| Attester slashings | Generate 50 attester slashings and check each validator client pair proposes a block containing at least one. Upstream runs this once during unfinality and once again after finality returns. | [attester-slashings.yaml](attester-slashings.yaml) | `InsertDoubleAttestationIntoPool` (#33), `ValidatorsSlashed` (#35), `SlashedValidatorsLoseBalance` (#36) |
| Proposer slashings | Generate 50 proposer slashings and check each validator client pair proposes a block containing at least one. Upstream runs this once during unfinality and once again after finality returns. | [proposer-slashings.yaml](proposer-slashings.yaml) | `InsertDoubleBlockIntoPool` (#34) |
| Finality recovery | Exit 150 validators and wait until the chain is finalizing again with at most 4 unfinalized epochs. | [finality-recovery.yaml](finality-recovery.yaml) | `FinalizationOccurs` (#20) |
| Cleanup | Exit all lifecycle validators and generate BLS changes for all lifecycle validators. | [cleanup.yaml](cleanup.yaml) | — (teardown) |

## Important caveats

These shards preserve the lifecycle test's operation-generation and
block-inclusion behavior. They still do not fully replace Prysm's richer result
evaluators unless extended with extra `check_consensus_validator_status` or
state/balance checks.

In particular:

- BLS, exit, and slashing shards prove the operation was accepted into a block,
  not that every affected validator reached the final expected state.
- The default `validatorMnemonic` is the lifecycle test mnemonic, not the
  default ethereum-package genesis mnemonic. Run the deposit shard first, or
  override `validatorMnemonic` and index ranges to target already-active keys.
- `validatorPairNames` is normally injected by ethereum-package's Assertoor
  config from participants with validator clients. If running outside
  ethereum-package, set it explicitly.
- Defaults mirror the upstream large lifecycle test. For a faster smoke shard,
  reduce `limitTotal`, `indexCount`, and timeouts, and keep index ranges
  non-overlapping.

## Example ethereum-package args

```yaml
assertoor_params:
  tests:
    - file: ./lifecycle/deposits.yaml
    - file: ./lifecycle/bls-changes.yaml
      config:
        startIndex: 0
        indexCount: 100
        limitTotal: 10
```

For the post-recovery "during finality" pass from the upstream lifecycle test,
reuse the BLS/exit/slashing shards with the same later index ranges used in
the full test:

- BLS changes: `startIndex: 150`, `indexCount: 50`
- exits: `startIndex: 150`, `indexCount: 50`
- attester slashings: `startIndex: 200`, `indexCount: 50`
- proposer slashings: `startIndex: 250`, `indexCount: 50`
