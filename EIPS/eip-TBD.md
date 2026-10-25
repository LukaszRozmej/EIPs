---
eip: TBD
title: Batched state proof precompile
description: Verify many account and storage values against a block header in one call, independent of the state tree format
author: Lukasz Rozmej (@LukaszRozmej)
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2026-10-25
requires: 2935
---

## Abstract

This EIP adds a precompile, `STATE_PROOF`, that verifies a batch of account fields and storage values against a block header. The caller gives the header, one unordered set of trie nodes, and a list of queries. Each query is an address and a list of storage slots. The precompile checks that the header hashes to the given block hash, takes the state root from the header, and returns the requested values. The precompile selects the trie rules from the header timestamp, so the same call works before and after a change of the state tree format.

## Motivation

Bridges, rollups, storage-proof oracles and snapshot-based applications prove L1 state inside contracts. Today they do this with Merkle-Patricia trie (MPT) verifiers written in Solidity. These verifiers have three problems.

**1. Cost.** We measured two widely used verifiers on real mainnet `eth_getProof` data for USDC and WETH (16 balance slots each), compiled with solc 0.8.25 (optimizer on, 200 runs):

| Verifier | 1 slot, tx gas | 16 slots, tx gas per slot | Calldata share |
| --- | ---: | ---: | ---: |
| Optimism `SecureMerkleTrie` (contracts-bedrock 0.17.3), one call per slot | 620,000 – 656,000 | 635,000 – 646,000 | 18% |
| Optimism `SecureMerkleTrie`, account proof verified once | – | 358,000 – 372,000 | 15% |
| Polytope Labs `EthereumTrie` (1.1.0), deduplicated node set | 1,743,000 – 1,875,000 | 1,818,000 – 1,919,000 | 2 – 6% |

One mainnet slot needs 16–17 nodes and about 7 KB of proof. Most of the cost is EVM execution: RLP decoding, memory copies, memory expansion and Keccak. Calldata is a fifth of the cost or less. A native implementation removes the execution cost almost completely. With the gas schedule below, the same proofs cost:

| Same mainnet proofs | 1 slot | 16 slots, per slot |
| --- | ---: | ---: |
| Precompile gas | 1,956 – 2,031 | 698 – 709 |
| Input size | 7,504 – 7,784 bytes | 2,564 – 2,612 bytes |
| Transaction gas ([EIP-7623](./eip-7623.md) floor applies) | 318,580 – 329,870 | 103,041 – 105,027 |
| Marginal gas inside a transaction that is above the floor | 121,000 – 126,000 | 41,400 – 42,200 |

After this EIP, calldata is almost all of the cost. A transaction that only verifies a proof pays the EIP-7623 floor, because its execution gas is small. Compared with the best Solidity verifier, the transaction is 1.9–2.0 times cheaper for one slot and 3.5 times cheaper per slot for 16 slots. When the call is part of a transaction whose execution is above the floor, the reduction is 5 times for one slot and 8.5 times per slot for 16 slots. Smaller proofs, for example from the binary tree of EIP-8297, reduce the cost further without a change to this precompile.

**2. Repeated work.** Proofs from `eth_getProof` have one path per slot. A contract that proves many slots of one account sends and hashes the header, the account path and the top of the storage path again for each slot. Deduplication in Solidity does not help much, because the node search and the memory expansion grow faster than linearly (see the Polytope row).

**3. Tree migration.** A change of the state tree, for example to the Partitioned Binary Tree of [EIP-8297](./eip-8297.md), breaks every deployed MPT verifier. [EIP-8347](./eip-8347.md) lists on-chain bridge contracts and consumers of state proofs as the users who carry this cost. Each application must deploy a new verifier and move to it at a known block.

A precompile solves all three. Clients already have trie verification code, so the execution is native. The batched input lets a caller send and hash each node one time. The precompile, not the caller, knows the tree format, so contracts that use it keep working after a tree migration without any change.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Constants

| Name | Value |
| --- | --- |
| `FORK_TIMESTAMP` | TBD |
| `STATE_PROOF_ADDRESS` | TBD |
| `GAS_BASE` | 100 |
| `GAS_PER_NODE` | 15 |
| `GAS_PER_NODE_WORD` | 6 |
| `GAS_PER_QUERY` | 100 |
| `GAS_PER_SLOT` | 100 |

### Input

All integers are unsigned and big-endian. The input is the concatenation of:

```
block_hash      bytes32
header_length   uint32
header          bytes[header_length]          RLP-encoded block header
node_count      uint32
nodes           node_count × (uint32 length, bytes[length])
query_count     uint32
queries         query_count × (address bytes20, uint32 slot_count, slot_count × bytes32)
```

The input MUST be consumed exactly. Trailing bytes are an error.

### Execution

1. Decode the input. If the input is malformed, fail.
2. If `keccak256(header) != block_hash`, fail.
3. Decode `header` as an RLP list. Its first 12 items MUST be byte strings. Let `state_root` be item 3, which MUST be 32 bytes, and `timestamp` be item 11. Otherwise, fail.
4. Select the state tree rules for `timestamp`, as defined in [State tree selection](#state-tree-selection).
5. Build a node pool: a map from the hash of each node, as defined by the selected tree rules, to the node. Duplicate and unused nodes are allowed.
6. For each query, in input order, look up the account for `address` from `state_root`. Then look up each slot. Use only nodes from the pool. If a node needed for a lookup is not in the pool, or a node is malformed, fail.
7. Return the output.

A failure consumes all gas given to the call, and the call returns no data. The precompile MUST NOT read the state. Its result depends only on its input.

All RLP in this EIP MUST be canonical: a single byte below `0x80` is not encoded as a string, the long forms are used only for lengths above 55, and lengths have no leading zero bytes. Each RLP item MUST fill the bytes that hold it exactly.

### State tree selection

- If `timestamp` is before the activation of [EIP-8347](./eip-8347.md), use the [MPT rules](#mpt-rules).
- Otherwise, use the [binary tree rules](#binary-tree-rules).

Under EIP-8347, every header's `stateRoot` commits to the whole state with exactly one tree, so no extra input is needed at the boundary. If a fork instead adopts a conversion in which `stateRoot` covers only part of the state, such as [EIP-7612](./eip-7612.md) with [EIP-7748](./eip-7748.md), the precompile MUST fail for headers whose `timestamp` is in the conversion window. That fork MAY specify a consensus-committed source for the base tree root, its deletions and the conversion progress, and extend this EIP to use them.

A fork that later changes the state tree again MUST define, for its tree: the node encoding and node hash used to build the pool, the account and slot lookups, and the absence rule. Absent accounts and slots MUST give zero values. The input and output formats do not change.

### MPT rules

- The node hash is `keccak256(node)`.
- The account key is `keccak256(address)`. The account value is `rlp([nonce, balance, storage_root, code_hash])`.
- The storage key is `keccak256(slot)`. The storage value is the RLP-encoded big-endian integer without leading zeros, of at most 32 bytes.
- A lookup of a 32-byte `key` from `root` works as follows:
  1. If `root` is the empty trie root `keccak256(0x80)`, the key is absent.
  2. Let `node = pool[root]` and `i = 0`, where `i` counts the key nibbles used.
  3. A node with 17 items is a branch. If `i = 64`, fail. Otherwise the next reference is item `key[i]`, and `i` increases by 1.
  4. A node with 2 items is a leaf or an extension. Item 0 is a hex-prefix path. Its first nibble is the flag: 0 or 1 for an extension, 2 or 3 for a leaf. Any other flag fails, and with an even flag the second nibble MUST be 0. If the path does not equal the next nibbles of the key, the key is absent. Otherwise `i` increases by the path length.
     - **Leaf:** `i` MUST now be 64. Item 1 is the value.
     - **Extension:** the path MUST NOT be empty, and `i` MUST be less than 64. Item 1 is the next reference.
  5. Any other item count fails.
  6. A reference that is an empty string means the key is absent. A 32-byte string is the hash of the next node, which MUST be in the pool. An RLP list is an embedded node: a node whose encoding is shorter than 32 bytes is embedded in its parent, and is not looked up in the pool. Any other reference fails.
- An absent account gives zero for all its fields and slots. An absent slot gives zero. If `storage_root` is the empty trie root, all slots are zero, and the lookup needs no nodes.

### Binary tree rules

These rules apply to the Partitioned Binary Tree of [EIP-8297](./eip-8297.md). `H`, `get_tree_key_for_basic_data`, `get_tree_key_for_code_hash`, `get_tree_key_for_delegation` and `get_tree_key_for_storage_slot` are defined there, and `address` is converted with `address20_to_address32`.

- A pool node is a hash preimage: `0x00 || key || value` for a leaf, or `0x01 || encode_bit_prefix(prefix) || left_hash || right_hash` for a branch. Its pool key is `H(node)`. `value`, `left_hash` and `right_hash` are 32 bytes.
- A lookup of `key` from `root` works as follows:
  1. If `root` is `0x00 * 32`, the key is absent.
  2. Let `node = pool[root]` and `d = 0`, where `d` counts the key bits used.
  3. **Leaf:** if the leaf's `key` equals the query key, the value is the leaf's `value`. Otherwise the key is absent.
  4. **Branch:** if `prefix` does not equal the key bits `d .. d + len(prefix) - 1`, the key is absent. If the key has no bit at `d + len(prefix)`, fail. Otherwise that bit selects `left_hash` (0) or `right_hash` (1) as the next node, and `d` increases by `len(prefix) + 1`.
  5. A node with any other tag or length fails.
- An account exists if its `CODE_HASH_LEAF_KEY` or its `DELEGATION_LEAF_KEY` leaf exists. `nonce` and `balance` come from the `BASIC_DATA_LEAF_KEY` value. `code_hash` is the `CODE_HASH_LEAF_KEY` value or, for a delegated account, `keccak256` of the first 23 bytes of the `DELEGATION_LEAF_KEY` value.
- Slot `s` is the value at `get_tree_key_for_storage_slot(address, s)`, or zero if it is absent.

### Output

For each query, in input order, the output has `4 + slot_count` words of 32 bytes:

```
exists      uint256    1 if the account exists, else 0
nonce       uint256
balance     uint256
code_hash   bytes32    0 if the account does not exist
values      slot_count × bytes32
```

### Gas

```
gas = GAS_BASE
    + GAS_PER_NODE_WORD × ceil(header_length / 32)
    + Σ over nodes   (GAS_PER_NODE + GAS_PER_NODE_WORD × ceil(length / 32))
    + Σ over queries (GAS_PER_QUERY + GAS_PER_SLOT × slot_count)
```

The gas is computed from the input before execution. If the call has less gas, it fails. If the input is malformed, the call fails and consumes all gas.

## Rationale

### Batched input

A single-slot API repeats the header, the account path and the top of the storage path for each slot. With the batched input, each node is sent and hashed one time, and each extra slot costs only its own lower storage nodes. On mainnet, 16 USDC slots need 107 unique nodes against 267 nodes for 16 separate proofs. The gas formula charges per node, not per slot, so the caller gets the full saving.

The saving grows under EIP-8297. Account fields and slots 0 to 63 share one stem, and slots in the same group of 256 share a stem too. A batch of related slots then shares almost its whole path.

### Unordered node pool

The node pool has no order and no structure. It has the same shape as the execution witness that clients build for stateless validation, so clients can use the same code to build and check it. A caller can merge several `eth_getProof` responses by removing duplicate nodes.

### Header in the input, not a state root

The precompile takes the header and checks it against the block hash, so a contract does not have to parse the header, whose format changes with forks. Item 3 (`stateRoot`) and item 11 (`timestamp`) have the same position in all header versions, and EIP-8347 keeps them in place.

A separate pure "verify against a state root" call is more general. For example, it can check a rollup output root that uses the same tree. But it requires contracts to link two calls correctly. This EIP specifies only the combined call. A later EIP can add a call with a state root as input if there is demand.

### Pure precompile

The precompile does not read the state, so it does not check that `block_hash` is canonical. The caller checks this with `BLOCKHASH` or with the history contract of [EIP-2935](./eip-2935.md). A pure precompile is easy to test, has no warm or cold access cost, and gives the same result in every context.

### Tree selection by timestamp

The header timestamp defines which tree produced its state root. Contracts that use this precompile do not change when the state tree changes. Clients keep MPT verification after the migration, which is small code that they already need to serve proofs for history.

The rule depends on EIP-8347, where every header commits to the whole state with one tree. Under an overlay conversion, part of the state is only in a frozen MPT whose root no header carries. A pure precompile cannot take that root from the caller safely, and the overlay rules for deletions are not specified. So this EIP fails in a conversion window instead of guessing.

### No storage root in the output

The output has only values that every planned tree supports. The EIP-8297 tree has no storage root per account, so the output does not include one.

### Failure consumes all gas

This is the same behavior as other recent precompiles. A malformed or incomplete proof is a caller error, and the caller can check the proof off-chain before it sends the transaction.

### Gas calibration

The gas has one term for each kind of work, so that the cost follows the work for every shape of input:

- `GAS_PER_NODE_WORD` pays for Keccak over the header and the nodes. It is the same as the per-word cost of the `KECCAK256` opcode.
- `GAS_PER_NODE` pays for the insertion of a node into the hash map.
- `GAS_PER_QUERY` pays for the hash of the address, the walk of the account trie and the decoding of the account.
- `GAS_PER_SLOT` pays for the hash of the slot and the walk of the storage trie.

We measured the Nethermind reference implementation (BenchmarkDotNet, .NET 10, one desktop x64 core) on the test vectors. A least-squares fit of time to the gas terms gives 54 ns per word, 120 ns per node, 780 ns per query and 410 ns per slot. On the same machine, the Keccak precompile runs at 99 MGas/s for a 32-byte input and at 130 MGas/s for a 136-byte input. The constants make every test vector run at 126–200 MGas/s, so the precompile is not cheaper per unit of time than Keccak.

`GAS_PER_SLOT` and `GAS_PER_QUERY` have a margin for deeper paths than the mainnet vectors. A deeper path needs keys with a longer common prefix, and each nibble of prefix needs 16 times more work to find. The vectors do not include such paths, so the margin is an estimate that a benchmark of deep paths must confirm. Inputs of many small nodes cost 21 gas for each node and run at about 120 MGas/s.

Other clients must confirm these values before the EIP can move to Review.

### Related work

- [EIP-7545](./eip-7545.md) (Stagnant) proposes a pure, versioned proof verification precompile for MPT and Verkle multiproofs. It returns only success or failure, takes a state root instead of a header, and lets the caller choose the version. This EIP returns the proven values, binds them to a block hash, and selects the tree from the header.
- [RIP-7728](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7728.md) (L1SLOAD) lets an L2 read L1 storage through its sequencer's view of the latest L1 block. It reads state and verifies no proof.
- [EIP-8272](./eip-8272.md) stores roots that applications write. It does not verify state proofs, and it can store roots that this precompile then uses through a later state-root variant.

## Backwards Compatibility

This EIP adds a new precompile. It does not change existing behavior.

## Test Cases

The test vectors are in [`test-vectors.json`](../assets/eip-TBD/test-vectors.json). Each vector has `Name`, `Input`, `Expected` (hex), and `Gas` for a successful call or `ExpectedError` for a failure. The vectors cover:

- many accounts with many slots, including an absent account, an absent slot, and an account with the empty storage root;
- an account without slots, the same account in two queries, and no queries;
- duplicate and unused nodes in the pool;
- embedded storage leaves, from two slots whose hashed keys share 9 leading nibbles;
- a missing node, a tampered node, a header that does not hash to `block_hash`, a header that is not an RLP list, trailing bytes, and truncated input;
- real mainnet headers with `eth_getProof` data for USDC (block 26268372) and WETH (block 26268373), with 1 slot and with 16 slots.

The expected outputs of the synthetic cases come from the state model that built the trie, and those of the mainnet cases come from the `eth_getProof` values. Vectors for the binary tree rules are TBD, until EIP-8297 fixes its hash function.

## Reference Implementation

A reference implementation of the MPT rules is in Nethermind, on the branch `feature/state-proof-precompile`: `StateProofPrecompile.cs`, with the tests and the vector generator in `StateProofPrecompileTests.cs`.

## Security Considerations

- **Canonical block.** The precompile proves only that a value is in the state of the block with hash `block_hash`. It does not prove that the block is canonical. A caller that does not check `block_hash` against `BLOCKHASH` or EIP-2935 accepts state from any header, including one that the caller made. Documentation and libraries for this precompile must show this check.
- **Denial of service.** The work is linear in the input size, and the gas is computed from the input before execution. Calldata cost limits the input size.
- **Absence proofs.** An implementation must accept a proof of absence only if the path ends in a node that excludes the key. An implementation that returns zero when it does not find a node gives false absence results. The rule "fail if a needed node is not in the pool" prevents this.
- **Reorgs.** A value proven against a recent block can change if that block is removed by a reorg. Callers that need final values must use finalized blocks.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
