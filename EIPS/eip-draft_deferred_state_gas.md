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
requires: 150, 2200, 2780, 7623, 7702, 7825, 7928, 8037, 8038
---

## Abstract

This proposal replaces the reservoir model of [EIP-8037](./eip-8037.md) with a single settlement at the end of the transaction. State-creating operations do not take state-gas from `gas_left`. They only increase a counter of the transaction's net state-gas, which is part of the state that a frame's rollback undoes. When execution ends, the counter is paid from the gas that execution did not use. If that gas is not sufficient, the transaction fails as a whole. A call frame can create net state-gas only up to an allowance derived from the gas forwarded to it, so a caller still limits what a callee can cost. A new `STATEGAS` opcode returns the net state-gas used so far. The pricing (`CPSB`), the two-dimensional block accounting and the [EIP-7825](./eip-7825.md) cap on execution gas of [EIP-8037](./eip-8037.md) do not change. The execution-gas charge of each state-creating operation is not lower than its cost before [EIP-8037](./eip-8037.md).

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

### Parameters

| Parameter | Value |
|---|---|
| `LEGACY_GAS_PER_STATE_BYTE` | 200 |
| `STATEGAS` opcode | `0x4c` (provisional) |
| `STATEGAS` gas cost | 2 |

`LEGACY_GAS_PER_STATE_BYTE` is `GAS_CODE_DEPOSIT` before [EIP-8037](./eip-8037.md), the lowest gas per new state byte of any operation before [EIP-8037](./eip-8037.md).

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
- If `state_gas_used` becomes greater than `state_gas_limit`, the transaction fails at once with a settlement failure. Execution stops: no further instruction is executed in any frame.
- The account-creation charge of a `CALL*` that transfers value to a non-existent account is added when the transfer is made. If the target has no code, nothing in the call can undo the transfer, so an implementation can apply the charge without a child frame. The charge is still undone when a frame that contains the `CALL*` rolls back.

The `GAS` opcode returns `gas_left`. The 63/64 rule of [EIP-150](./eip-150.md) and the `SSTORE` stipend check of [EIP-2200](./eip-2200.md) apply to `gas_left`, as before [EIP-8037](./eip-8037.md).

### State-gas allowance of call frames

Each call frame has a `state_gas_allowance` and records `state_gas_used_at_entry`, the value of `state_gas_used` when the frame starts. The net state-gas of a frame is `state_gas_used - state_gas_used_at_entry`. It includes the state-gas of its successful descendants and can be negative.

- The top-level frame has no allowance. Only `state_gas_limit` applies to it.
- A frame that a `CALL*`, `CREATE` or `CREATE2` starts with `child_gas` of `gas_left` (after the 63/64 rule and including the call stipend) gets:

  ```python
  parent_net_state_gas = state_gas_used - parent.state_gas_used_at_entry
  child.state_gas_allowance = min(
      parent.state_gas_allowance - parent_net_state_gas,  # no limit if the parent has none
      child_gas * CPSB // LEGACY_GAS_PER_STATE_BYTE)
  ```

- After each state-gas charge, including the code-deposit charge of a create frame, the frame that is charged checks its net state-gas. If it is greater than its `state_gas_allowance`, the frame halts exceptionally, as on out-of-gas: its state changes are rolled back and its `gas_left` is consumed. The caller continues, and can detect the failure as for any failed call.
- The account-creation charge of a `CALL*` with value and of a `CREATE`/`CREATE2` is charged to the calling frame, before the child frame starts. The code-deposit charge is charged to the create frame.

The check against `state_gas_limit` is applied first. When both checks fail, the transaction fails.

### `STATEGAS` opcode

`STATEGAS` takes no input. It pushes `state_gas_used`, the net state-gas of the transaction so far, without `committed_state_gas`. It costs 2 gas. The value can decrease, because refills and rollbacks decrease `state_gas_used`.

### Execution-gas of state-creating operations

The execution-gas charge of a state-creating operation is not lower than the cost of the same operation before [EIP-8037](./eip-8037.md):

| Operation | Execution-gas charge |
|---|---|
| `SSTORE` that creates a slot (original and current value zero, new value non-zero) | access cost + `max(STORAGE_WRITE, 20000)` |
| `CALL` with value to a non-existent account, `SELFDESTRUCT` that creates an account | charge of [EIP-8038](./eip-8038.md) + `25000` |
| `CREATE`, `CREATE2` | `max(CREATE_ACCESS, 32000)` instead of `CREATE_ACCESS` |

The constants `20000`, `25000` and `32000` are the values of `GAS_STORAGE_SET`, `GAS_NEW_ACCOUNT` and `GAS_CREATE` before [EIP-8037](./eip-8037.md). These charges are execution-gas. The code-deposit charge and the pre-execution charges are not changed.

An `SSTORE` that restores a slot to an original value of zero (original value zero, current value non-zero, new value zero) adds `max(STORAGE_WRITE, 20000)` to the refund counter, instead of the `STORAGE_WRITE` of [EIP-8038](./eip-8038.md). Thus a slot that is set and cleared in the same transaction costs only its access, as before [EIP-8037](./eip-8037.md). The refund cap applies.

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

A settlement failure is not an out-of-gas condition of any frame. Clients SHOULD report it in traces as a failure of the top-level frame with its own error, for example `state gas settlement failed`.

The refund cap, the [EIP-7623](./eip-7623.md) calldata floor, the receipt `cumulative_gas_used` and the block-level accounting apply as in [EIP-8037](./eip-8037.md), with `tx_state_gas` from above. The unused part of `tx.gas` is returned to the sender.

### Repricing surcharge (extension)

This section is a separable extension. It applies the settlement to the execution-gas increases of [EIP-8038](./eip-8038.md), so that deployed contracts see no repricing inside call frames.

A transaction-level counter `surcharge_used` starts at zero. It only increases: a revert or an exceptional halt does not decrease it, because the work was done.

- Each charge to `gas_left` that [EIP-8038](./eip-8038.md) raises is charged at its value before [EIP-8038](./eip-8038.md), and the increase is added to `surcharge_used`. For example, the first change of an existing slot takes the access cost plus 2,800 from `gas_left` and adds `STORAGE_WRITE - 2800` to `surcharge_used`. A cold account access takes 2,600 and adds `COLD_ACCOUNT_ACCESS - 2600`. A `CALL` with value takes 9,000 and adds `CALL_VALUE - 9000`.
- Charges that [EIP-8038](./eip-8038.md) lowers, and the state-creating operations of this proposal, are not changed by this section. Intrinsic gas and the refund counter use the values of [EIP-8038](./eip-8038.md) directly, because contracts cannot observe them during execution.
- The check during execution becomes `state_gas_used + surcharge_used > state_gas_limit`.
- Settlement charges `state_gas_used + surcharge_used`:

  ```python
  exec_gas_used = gas_left_initial - gas_left
  settlement_gas = state_gas_used + surcharge_used
  if settlement_gas > state_gas_limit - exec_gas_used:
      settlement_failure()
  tx_gas_used_before_refund = intrinsic_gas + exec_gas_used + surcharge_used + committed_state_gas + state_gas_used
  ```

- On a settlement failure, the sender pays `min(tx.gas, intrinsic_gas + committed_state_gas + gas_left_initial + surcharge_used)`.
- `surcharge_used` is execution-gas in the block accounting, also when settlement fails: in that case, `tx_execution_gas = max(intrinsic_gas + gas_left_initial + surcharge_used, calldata_floor_gas_cost)`.
- The transaction-validity check for the execution dimension becomes `min(TX_MAX_GAS_LIMIT, tx.gas) * 7 // 2 <= execution_gas_available`. A charge is raised at most 3.5 times, by the first change of a warm existing slot (2,900 before, 10,100 after), so this is the most execution-gas a transaction can add to the block.
- `STATEGAS` returns `state_gas_used + surcharge_used`, the gas that settlement will charge so far.

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

Execution stops at once when the check fails. Execution that continued could not change the result, because the transaction fails and `gas_left` is consumed in both cases. But it would access more accounts and slots, and these accesses would enter the [EIP-7928](./eip-7928.md) block access list. Stopping at once makes the block access list of a failed transaction unambiguous.

### `gas_left` includes gas that settlement can need

Under [EIP-8037](./eip-8037.md), a state-gas charge without a reservoir lowers `gas_left` at once. With this proposal, `gas_left` is not lowered, and the gas stays available for execution until settlement. Two effects follow:

- `GAS` returns more than under [EIP-8037](./eip-8037.md) after state creation, and a transaction with `tx.gas <= TX_MAX_GAS_LIMIT` cannot have a small `gas_left` and a state budget at the same time.
- A frame that forwards all its gas forwards gas that settlement can need later. For example, a child `CREATE` that fails consumes 63/64 of a `gas_left` that the account-creation charge did not lower, so it consumes about 63/64 of `STATE_BYTES_PER_NEW_ACCOUNT × CPSB` more gas than under [EIP-8037](./eip-8037.md).

Both effects match the gas semantics before [EIP-8037](./eip-8037.md), where all gas was available for execution. A sender who wants a frame to keep gas for settlement must supply it in `tx.gas`.

### State-gas allowance

Before [EIP-8037](./eip-8037.md), the gas that a caller forwards limited everything that a callee could cost, including state: the cheapest new state byte cost `LEGACY_GAS_PER_STATE_BYTE` (200) gas, as code deposit. Contracts that call untrusted code rely on this. Without an allowance, settlement removes this limit: a callee can deploy a large contract for little execution-gas, and the debt can fail the whole transaction, which no caller can catch.

The allowance restores the limit with the only input that deployed contracts give, the forwarded gas. A frame with `g` gas can create at most `g / LEGACY_GAS_PER_STATE_BYTE` new state bytes, which is what `g` gas could buy before [EIP-8037](./eip-8037.md). Thus:

- The allowance never fails a call that the same gas could complete before [EIP-8037](./eip-8037.md). Gas budgets that were sufficient stay sufficient.
- A callee can create at most about `7.65 × g` of state-gas (`CPSB / 200`). A sender that executes untrusted code can make a settlement failure impossible: it sets `tx.gas` to at least the execution-gas plus `CPSB / LEGACY_GAS_PER_STATE_BYTE` times the gas it forwards to untrusted code. This needs only balance, because the unused gas is returned.
- The allowance depends only on the forwarded gas. `tx.gas` does not change it, so the sender cannot use it to make a call fail that would succeed otherwise, beyond what the forwarded gas already allows.
- When a frame exceeds its allowance, only that frame fails. The caller handles it as an out-of-gas of the callee.

The child allowance is also limited by the parent's remaining allowance, so the net state-gas of a frame and all its descendants stays within the frame's allowance. Because the parent does not run while the child runs, only the frame that is charged must be checked.

The allowance limits deployments from a call frame: a frame with 16.7M gas can deploy about 83 kB of code, as before [EIP-8037](./eip-8037.md). This is sufficient for a contract of the maximum size of [EIP-7954](./eip-7954.md). The top-level frame has no allowance, so a deployment from the transaction itself, or from a factory that the transaction calls directly, is limited only by `tx.gas`.

A lower ratio, for example from the 312 gas per byte of a new slot before [EIP-8037](./eip-8037.md), would expose senders less. But it would fail deployments from call frames that the same gas could complete before [EIP-8037](./eip-8037.md).

### `STATEGAS`

`gasleft()` differences do not include state-gas, so contracts that bill a sub-call for its work, such as an [ERC-4337](./eip-4337.md) `EntryPoint`, cannot bill state-gas. `STATEGAS` lets new contracts measure it: the state-gas of a sub-call is the difference between two `STATEGAS` reads around it. The opcode exposes a value that the transaction already has. It does not change the cost or the result of any other operation.

### Repricing surcharge

A repricing that raises execution-gas can make a fixed gas budget of a deployed contract too small. For example, [EIP-8038](./eip-8038.md) raises the first change of a cold existing slot from 5,000 to 12,100, so an ERC-20 transfer to an existing holder costs about 14,200 more. Where a contract records such a failure permanently, the repricing causes a permanent loss. The surcharge keeps the costs inside call frames at their values before [EIP-8038](./eip-8038.md), and the sender still pays the new prices through settlement.

- **Work per transaction.** `gas_left` is still capped at `TX_MAX_GAS_LIMIT - intrinsic_gas`, but in prices before [EIP-8038](./eip-8038.md). The most work that one transaction can do is therefore the same as before [EIP-8038](./eip-8038.md), up to 3.5 times `TX_MAX_GAS_LIMIT` in [EIP-8038](./eip-8038.md) prices for a transaction of only warm storage updates. The extension gives up the tighter limit per transaction that [EIP-8038](./eip-8038.md) would add. It keeps the limit per block: `surcharge_used` counts as execution-gas of the block.
- **No allowance needed.** The surcharge of a charge is at most 2.5 times its value inside the frame. So the surcharge of a call frame is at most 2.5 times the gas it was given, and a sender can make settlement safe by its choice of `tx.gas`.
- **Not undone by a rollback.** The surcharge pays for work that was done, so it stays when a frame reverts or halts, as consumed gas does.
- **Future repricings.** The same rule can apply to later repricings, so that deployed contracts never see a repricing inside a call frame.

### Authorizations are paid before execution

[EIP-7702](./eip-7702.md) authorizations stay applied when execution fails. Their state-gas must therefore be paid even if settlement fails, so it is taken from `tx.gas` before `gas_left` is set.

### What is removed

Compared with [EIP-8037](./eip-8037.md), this proposal removes the reservoir, the four frame-level counters, the last-in, first-out refill order, the merge step, the restore and commit rules, the partial reservoir of system calls, and the rule that the `GAS` opcode excludes the reservoir. It adds one journaled counter, an allowance per frame, two checks at each state-gas charge, one check at the end of the transaction, and the `STATEGAS` opcode.

## Backwards Compatibility

This is a backwards-incompatible change of the gas model that requires a scheduled network upgrade. It is intended to be activated together with [EIP-8037](./eip-8037.md), instead of the reservoir model.

- `GAS` and the forwarding of gas to calls behave as before [EIP-8037](./eip-8037.md). The difference between two `GAS` reads is never negative.
- `gasleft()` differences do not include state-gas. Contracts that use them to charge a user for the work of a sub-call, for example the [ERC-4337](./eip-4337.md) `EntryPoint`, do not charge for state-gas. This is the same as for state-gas taken from the reservoir under [EIP-8037](./eip-8037.md), and similar to fees that rollups charge outside of gas. New contracts can use `STATEGAS`.
- A call frame that creates more state than its forwarded gas could buy before [EIP-8037](./eip-8037.md) fails with an exceptional halt.
- `STATEGAS` is a new opcode. `0x4c` was an undefined instruction before.
- A transaction can fail at settlement after its execution completed. Wallets must use `eth_estimateGas`, which includes the settlement.
- `GAS` returns more than under [EIP-8037](./eip-8037.md) after state creation, and a frame that forwards all its gas forwards gas that settlement can need (see [`gas_left` includes gas that settlement can need](#gas_left-includes-gas-that-settlement-can-need)).

## Security Considerations

### Cost of untrusted code to the sender

The allowance limits the state-gas that a sub-call can create to `CPSB / LEGACY_GAS_PER_STATE_BYTE` (about 7.65) times the gas forwarded to it. The sender pays for this state-gas. A contract that bills sub-calls by `gasleft()` differences, such as a deployed [ERC-4337](./eip-4337.md) `EntryPoint`, does not bill it. A bundler or relayer that uses such a contract must simulate the transaction and charge for the state-gas in another way, for example through `preVerificationGas`, or accept the cost. New contracts can bill it with `STATEGAS`. Under [EIP-8037](./eip-8037.md) the same applies when the transaction has a reservoir, without a limit per sub-call.

### Untrusted code cannot fail the whole transaction

Without the allowance, a sub-call could create more state-gas than the transaction can pay, for example by deploying a large contract with little execution-gas. The settlement would then fail the whole transaction, and no caller could catch it: one bad user operation could make an [ERC-4337](./eip-4337.md) bundle fail. With the allowance, a sub-call that exceeds its limit fails alone, and its caller continues. A sender that sets `tx.gas` to at least its execution-gas plus `CPSB / LEGACY_GAS_PER_STATE_BYTE` times the gas forwarded to untrusted code cannot get a settlement failure from that code.

### Wasted execution

A transaction that fails at settlement pays for all of its execution-gas, as a transaction that runs out of gas does. Its new state is not written. Nodes and builders are paid for all work they do.

### Unpaid state during execution

The check against `state_gas_limit` limits the new state that a transaction can create during execution to the state that `tx.gas` could pay for.

### Underpaid work of failed transactions (repricing surcharge)

With the repricing surcharge, a transaction can do work whose surcharge it cannot pay: it uses all of `gas_left` on storage updates, and settlement fails. The sender then pays all of `tx.gas`, which covers the work at the prices before [EIP-8038](./eip-8038.md) but not at the new prices. The block still counts the full work, so the block limit is kept. But such a transaction raises the base fee by more than it pays for, by at most 3.5 times. The work and the price are the same as before [EIP-8038](./eip-8038.md), and the state changes of the transaction are rolled back.

### Block building

A builder learns the gas that a transaction uses only after it executes the transaction, as before this proposal. Settlement does not change this, and every included transaction pays, also when settlement fails. Two things change:

- The validity check before execution must reserve the most that a transaction can use: `tx.gas` in the state dimension, as in [EIP-8037](./eip-8037.md), and, with the repricing surcharge, 3.5 times `min(TX_MAX_GAS_LIMIT, tx.gas)` in the execution dimension. Near the end of a block, a transaction can therefore be rejected although its actual use would fit. The loss is at most one reservation per block and dimension.
- A transaction with a large `tx.gas` for state, such as a large deployment, needs that much free state capacity. It fits only in a block that has not used much state yet.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
