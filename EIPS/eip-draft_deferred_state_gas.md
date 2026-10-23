---
eip: TBD
title: Deferred State-Gas Settlement
description: Settle state-gas once at the end of the transaction instead of charging it inside call frames
author: Łukasz Rozmej (@LukaszRozmej)
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2026-10-23
requires: 150, 2200, 2780, 7623, 7702, 7825, 8037, 8038
---

## Abstract

This proposal replaces the reservoir model of [EIP-8037](./eip-8037.md) with a single settlement at the end of the transaction. State-creating operations do not take state-gas from `gas_left`. They only increase a counter of the transaction's net state-gas, which is part of the state that a frame's rollback undoes. When execution ends, the counter is paid from the gas that execution did not use. If that gas is not sufficient, the transaction fails as a whole. The pricing (`CPSB`), the two-dimensional block accounting and the [EIP-7825](./eip-7825.md) cap on execution gas of [EIP-8037](./eip-8037.md) do not change. The execution-gas charge of each state-creating operation is not lower than its cost before [EIP-8037](./eip-8037.md).

## Motivation

[EIP-8037](./eip-8037.md) meters state-gas separately from execution-gas, so that the [EIP-7825](./eip-7825.md) cap applies to execution only and large contracts can still be deployed. Because a transaction has a single gas limit, it splits `tx.gas` into `gas_left` and a `state_gas_reservoir`. State-gas is taken from the reservoir first and from `gas_left` after that, and refills and rollbacks return state-gas to `gas_left`.

This puts state-gas inside the call frames, and makes the gas that a frame can see and use depend on things the frame does not control:

1. **Spent gas comes back.** Clearing a slot that was zero at the start of the transaction returns state-gas to `gas_left` at once. A frame that reverts returns to its caller the state-gas it paid from `gas_left`. Contracts that inspect `gasleft()` after a failed call to tell an out-of-gas failure from a logic error can then draw the wrong conclusion.
2. **The sender sets the in-frame price of state creation.** With a reservoir (`tx.gas > TX_MAX_GAS_LIMIT`), a frame pays only the execution part of a new slot (12,100 gas when cold, compared with 22,100 before). A failure handler that used to run out of gas can complete. Without a reservoir, the frame pays the full 110,020 gas, and a sub-call budget that was sufficient before can run out of gas. Both directions are available to an attacker who sends the transaction.
3. **The accounting is complex.** The reservoir model needs four counters, three of them frame-local, a last-in, first-out refill order, a merge step for successful children, a separate restore rule for rollbacks, and a commit rule for [EIP-7702](./eip-7702.md) authorizations.

All three come from charging state-gas inside frames. The amount of state a transaction creates is known only at the end, and it is a property of the transaction, not of a frame. This proposal settles it once, at the end.

## Specification

The parameters and the state-gas amounts of [EIP-8037](./eip-8037.md) apply: `CPSB`, `STATE_BYTES_PER_STORAGE_SET`, `STATE_BYTES_PER_NEW_ACCOUNT`, `STATE_BYTES_PER_AUTH_BASE` and `TX_MAX_TOTAL_GAS_LIMIT`, and the operations listed in its sections "Gas accounting for SSTORE (opcode-level)" and "Gas accounting for new accounts". The transaction validity rules and the block-level accounting of [EIP-8037](./eip-8037.md) apply as specified there, with `tx_output.evm_state_gas_used` replaced by `tx_state_gas` below.

This proposal replaces the section "Transaction-level gas accounting (reservoir model)" of [EIP-8037](./eip-8037.md), including its subsections on halts, reverts and successful children. `state_gas_reservoir`, `state_gas_from_gas_left`, `state_gas_baseline` and `state_gas_committed` are removed.

### Pre-execution

During the pre-execution phase of [EIP-2780](./eip-2780.md):

```python
evm_gas = tx.gas - intrinsic_gas
committed_state_gas = <state-gas of the EIP-7702 authorizations>
assert committed_state_gas <= evm_gas  # otherwise the pre-execution phase fails, as in EIP-2780
gas_left = min(TX_MAX_GAS_LIMIT - intrinsic_gas, evm_gas - committed_state_gas)
gas_left_initial = gas_left
state_gas_limit = evm_gas - committed_state_gas
state_gas_used = 0
```

`committed_state_gas` is the state-gas of the pre-execution charges that a rollback of the top-level frame does not undo, which are the [EIP-7702](./eip-7702.md) authorization charges. It is paid before execution starts.

The account-creation charge of a contract-creation transaction is undone by a rollback of the top-level frame. It is added to `state_gas_used` when the top-level frame starts.

### State-gas during execution

`state_gas_used` is part of the state of the transaction. It is not part of `gas_left`:

- A state-gas charge listed in [EIP-8037](./eip-8037.md) adds its amount to `state_gas_used`. It does not change `gas_left`.
- A state-gas refill listed in [EIP-8037](./eip-8037.md) subtracts its amount from `state_gas_used`. It does not change `gas_left`.
- When a frame reverts or halts exceptionally, `state_gas_used` is set back to its value at the entry of the frame, together with the other state changes of the frame. The remaining `gas_left` of the frame is returned to the parent or consumed, as before [EIP-8037](./eip-8037.md).
- If `state_gas_used` becomes greater than `state_gas_limit`, the transaction fails at once with a settlement failure.

The `GAS` opcode returns `gas_left`. The 63/64 rule of [EIP-150](./eip-150.md) and the `SSTORE` stipend check of [EIP-2200](./eip-2200.md) apply to `gas_left`, as before [EIP-8037](./eip-8037.md).

### Execution-gas of state-creating operations

The execution-gas charge of a state-creating operation is not lower than the cost of the same operation before [EIP-8037](./eip-8037.md):

| Operation | Execution-gas charge |
|---|---|
| `SSTORE` that creates a slot (original and current value zero, new value non-zero) | access cost + `max(STORAGE_WRITE, 20000)` |
| `CALL` with value to a non-existent account, `SELFDESTRUCT` that creates an account | charge of [EIP-8038](./eip-8038.md) + `25000` |
| `CREATE`, `CREATE2` | `max(CREATE_ACCESS, 32000)` instead of `CREATE_ACCESS` |

The constants `20000`, `25000` and `32000` are the values of `GAS_STORAGE_SET`, `GAS_NEW_ACCOUNT` and `GAS_CREATE` before [EIP-8037](./eip-8037.md). These charges are execution-gas. The code-deposit charge and the pre-execution charges are not changed.

### Settlement

When the top-level frame ends:

```python
exec_gas_used = gas_left_initial - gas_left
if state_gas_used > state_gas_limit - exec_gas_used:
    settlement_failure()  # exec_gas_used = gas_left_initial, state_gas_used = 0
tx_state_gas = committed_state_gas + state_gas_used
tx_gas_used_before_refund = intrinsic_gas + exec_gas_used + tx_state_gas
```

A settlement failure is processed as an exceptional halt of the top-level frame: all state changes of the execution are rolled back, `gas_left` is consumed, the refund counter is set to zero and `state_gas_used` is set to zero. The [EIP-7702](./eip-7702.md) authorizations stay applied, and `committed_state_gas` stays charged. The receipt status is `0`.

The refund cap, the [EIP-7623](./eip-7623.md) calldata floor, the receipt `cumulative_gas_used` and the block-level accounting apply as in [EIP-8037](./eip-8037.md), with `tx_state_gas` from above. The unused part of `tx.gas` is returned to the sender.

### System calls

System calls get `30_000_000` `gas_left` and no settlement. Their state-gas is not limited and does not count toward the block. `SYSTEM_MAX_SSTORES_PER_CALL` is removed.

## Rationale

### The cause is the per-transaction execution cap

[EIP-7825](./eip-7825.md) caps a transaction at `TX_MAX_GAS_LIMIT` (16,777,216). With `CPSB = 1530`, a cap that also included state-gas would limit a deployment to about 7.5 kB. [EIP-8037](./eip-8037.md) therefore applies the cap to execution-gas only. Because a transaction has one gas limit, it must decide how to split `tx.gas` between the two dimensions before it knows how much of each the execution will use. The reservoir makes that decision at the start and corrects it during execution, which is the source of the frame-visible effects listed in the motivation.

At the end of the transaction, the split is known. Execution-gas is limited by `gas_left`, which starts at no more than `TX_MAX_GAS_LIMIT - intrinsic_gas`. State-gas is limited by what is left of `tx.gas`. No split has to be guessed, and nothing has to be corrected.

### Frames do not see state-gas

With this proposal, no amount that a contract can observe depends on state-gas:

- `gas_left` only decreases within a frame, and a call returns at most the unused gas of the child, as before [EIP-8037](./eip-8037.md).
- The gas that a frame needs for an operation does not depend on `tx.gas`. A sub-call budget that was sufficient for state creation before [EIP-8037](./eip-8037.md) is still sufficient for state creation, whoever sends the transaction.
- A transaction that does not supply enough gas for its state fails as a whole. A contract cannot catch this failure, so it cannot be used to make a contract record a failure of an operation that was given too little gas.

The repricing of [EIP-8038](./eip-8038.md) still changes the execution-gas of operations that do not create state. This proposal does not change that.

### Execution-gas floor for state creation

[EIP-8038](./eip-8038.md) charges a new slot 10,000 execution-gas plus access, because [EIP-8037](./eip-8037.md) moves the cost of the new state to state-gas. When the state-gas does not come from the frame, a frame could then create a slot for less gas than before (12,100 instead of 22,100 when cold). A contract that expects its failure handler to run out of gas could then record a failure it could not record before. The floor keeps every state-creating operation at least as expensive in execution-gas as before [EIP-8037](./eip-8037.md), so that any frame that ran out of gas before still runs out of gas.

The floor also limits how much state-gas a frame can create per unit of the gas it was given, to about 4 to 6 times for slots and accounts.

The floor is not applied to the code deposit. A floor of 200 gas per byte would again limit deployments by `TX_MAX_GAS_LIMIT`, which [EIP-8037](./eip-8037.md) intends to remove. A failure handler that deploys code is not a known pattern.

The floor increases the total cost of a new slot by 10,000 gas (about 9%) compared with [EIP-8037](./eip-8037.md). It is charged as execution-gas, so a transaction can create about as many slots as before [EIP-8037](./eip-8037.md) under [EIP-7825](./eip-7825.md).

### Failure at the end of the transaction

[EIP-8037](./eip-8037.md) rejects charging the account creation of `CREATE` after the create frame exits, because a successful deployment could then run out of gas at the frame exit. Settlement does not have this problem: it does not fail a frame, it fails the whole transaction, as an out-of-gas of the top-level frame does. `eth_estimateGas` finds the required `tx.gas` in the same way as for execution-gas.

### Bound on unpaid state during execution

`state_gas_used` is paid only at the end. Without a bound, a transaction could hold much more unpaid new state during execution than it can pay for, for example by deploying large contracts with little execution-gas. The check against `state_gas_limit` limits this to what `tx.gas` could pay for if the execution used no gas, which is the same limit as with the reservoir. The check uses a value that is fixed for the transaction, so it is simple to compute at every charge. The exact check is done at settlement.

### Authorizations are paid before execution

[EIP-7702](./eip-7702.md) authorizations stay applied when execution fails. Their state-gas must therefore be paid even if settlement fails, so it is taken from `tx.gas` before `gas_left` is set.

### What is removed

Compared with [EIP-8037](./eip-8037.md), this proposal removes the reservoir, the four frame-level counters, the last-in, first-out refill order, the merge step, the restore and commit rules, the partial reservoir of system calls, and the rule that the `GAS` opcode excludes the reservoir. It adds one journaled counter, a check at each state-gas charge against a fixed limit, and one check at the end of the transaction.

## Backwards Compatibility

This is a backwards-incompatible change of the gas model that requires a scheduled network upgrade. It is intended to be activated together with [EIP-8037](./eip-8037.md), instead of the reservoir model.

- `GAS` and the forwarding of gas to calls behave as before [EIP-8037](./eip-8037.md). The difference between two `GAS` reads is never negative.
- `gasleft()` differences do not include state-gas. Contracts that use them to charge a user for the work of a sub-call, for example the [ERC-4337](./eip-4337.md) `EntryPoint`, do not charge for state-gas. This is the same as for state-gas taken from the reservoir under [EIP-8037](./eip-8037.md), and similar to fees that rollups charge outside of gas.
- A transaction can fail at settlement after its execution completed. Wallets must use `eth_estimateGas`, which includes the settlement.

## Security Considerations

### Cost of untrusted code to the sender

The gas that a caller forwards to a sub-call does not limit the state-gas that the sub-call can create. For slots and accounts, the execution-gas floor limits it to about 4 to 6 times the forwarded gas. For code deposit, it is not limited by the forwarded gas: a sub-call can deploy a large contract with little execution-gas. The sender pays for this state-gas, up to `tx.gas`. Senders that execute untrusted code, such as bundlers and relayers, must simulate the transaction and include its state-gas in what they charge, or set `tx.gas` so that a settlement failure is acceptable to them. Under [EIP-8037](./eip-8037.md) the same applies when the transaction has a reservoir.

### Untrusted code can fail the whole transaction

Before this proposal, a sub-call with a limited gas budget cannot make the transaction of its caller fail: the sub-call runs out of gas, and the caller continues. With this proposal, a sub-call can create more state-gas than the transaction can pay, for example by deploying a large contract. The settlement then fails the whole transaction, and the caller cannot catch this. One bad user operation can make an [ERC-4337](./eip-4337.md) bundle fail, and the bundler pays for all of its execution-gas. Batch executors, keepers and relayers have the same problem. Under [EIP-8037](./eip-8037.md), the sub-call pays the state-gas from its own `gas_left` when the reservoir is exhausted, so only the sub-call runs out of gas.

This is an open issue of this proposal. A solution must let a caller limit the state-gas of a sub-call. For deployed contracts, the only available input is the gas that the caller forwards.

### Wasted execution

A transaction that fails at settlement pays for all of its execution-gas, as a transaction that runs out of gas does. Its new state is not written. Nodes and builders are paid for all work they do.

### Unpaid state during execution

The check against `state_gas_limit` limits the new state that a transaction can create during execution to the state that `tx.gas` could pay for.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
