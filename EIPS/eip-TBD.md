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
| Precompile gas | 2,196 – 2,286 | 818 – 833 |
| Input size | 7,505 – 7,785 bytes | 2,564 – 2,612 bytes |
| Transaction gas ([EIP-7623](./eip-7623.md) floor applies) | 318,620 – 329,910 | 103,044 – 105,029 |
| Marginal gas inside a transaction that is above the floor | 121,000 – 126,000 | 41,500 – 42,400 |

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
| `GAS_PER_TRIE_NODE` | 15 |

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
3. Decode `header` as an RLP list. Its first 12 items MUST be byte strings. Let `state_root` be item 3, which MUST be 32 bytes. Item 11 encodes `timestamp` as an unsigned big-endian integer of at most 8 bytes, with no leading zero bytes; zero MUST be the empty byte string. Otherwise, fail.
4. Select the state tree rules for `timestamp`, as defined in [State tree selection](#state-tree-selection).
5. Build a node pool: a map from the hash of each node, as defined by the selected tree rules, to the node. Duplicate and unused nodes are allowed.
6. For each query, in input order, look up the account for `address` from `state_root`. Then look up each slot. Use only nodes from the pool. If a node needed for a lookup is not in the pool, or a node is malformed, fail.
7. Return the output.

A failure consumes all gas given to the call, and the call returns no data. The precompile MUST NOT read the state. For a fixed executing fork, its result depends only on its input.

All RLP in this EIP MUST be canonical: a single byte below `0x80` is not encoded as a string, the long forms are used only for lengths above 55, and lengths have no leading zero bytes. Each RLP item MUST fill the bytes that hold it exactly.

### State tree selection

The executing block's fork selects the version of the precompile. Within that version:

- Before [EIP-8347](./eip-8347.md) is active in the executing block, use the [MPT rules](#mpt-rules) for every supplied `timestamp`, even if a future EIP-8347 activation is configured.
- Once EIP-8347 is active in the executing block, use the MPT rules for supplied timestamps before its activation timestamp, and the [binary tree rules](#binary-tree-rules) for supplied timestamps at or after it.

Scheduling or activating a later fork MUST NOT change the rules used to replay an earlier executing block. The supplied header need not be canonical or have a timestamp at or before the executing block, so clients MUST NOT select the precompile version from that header.

Under EIP-8347, every header's `stateRoot` commits to the whole state with exactly one tree, so no extra input is needed at the boundary. If a fork instead adopts a conversion in which `stateRoot` covers only part of the state, such as [EIP-7612](./eip-7612.md) with [EIP-7748](./eip-7748.md), the precompile MUST fail for headers whose `timestamp` is in the conversion window. That fork MAY specify a consensus-committed source for the base tree root, its deletions and the conversion progress, and extend this EIP to use them.

A fork that later changes the state tree again MUST define a new precompile version, preserving earlier versions for historical execution. It MUST define, for its tree: the node encoding and node hash used to build the pool, the account and slot lookups, and the absence rule. Absent accounts and slots MUST give zero values. The input and output formats do not change.

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
- Look up `CODE_HASH_LEAF_KEY` first. If it exists, the account exists and its value is `code_hash`; do not look up `DELEGATION_LEAF_KEY`. Otherwise, look up `DELEGATION_LEAF_KEY`. If it exists, the account exists and `code_hash` is `keccak256` of its first 23 bytes. If both are absent, return zero for all account fields and slots without further lookups.
- For an existing account, look up `BASIC_DATA_LEAF_KEY`; `nonce` and `balance` come from its value, or are zero if it is absent. Then, in input order, look up each slot `s` at `get_tree_key_for_storage_slot(address, s)`, returning its value or zero if it is absent. Every lookup starts from `state_root`.

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
input_gas = GAS_BASE
    + GAS_PER_NODE_WORD × ceil(header_length / 32)
    + Σ over nodes   (GAS_PER_NODE + GAS_PER_NODE_WORD × ceil(length / 32))
    + Σ over queries (GAS_PER_QUERY + GAS_PER_SLOT × slot_count)

gas = input_gas + GAS_PER_TRIE_NODE × visited_nodes
```

Deduct `input_gas` before execution. Before resolving and examining each trie node during a lookup, deduct `GAS_PER_TRIE_NODE`. If either deduction exceeds the remaining gas, fail immediately. Malformed input also fails. Every failure consumes all gas supplied to the call and returns no data.

`visited_nodes` counts the nodes traversed by the lookups specified above, including roots, leaves, extensions, branches and embedded MPT nodes. Each lookup starts at its specified root. A shared node is counted again on each visit, including visits from repeated account or slot queries. An empty trie root or empty MPT child reference establishes absence without visiting a node. Pool nodes that are not traversed incur only their input charge.

Implementations MAY cache decoded nodes or lookup results, but MUST charge for the same logical node visits as the specified traversal, independent of cache contents. A cached lookup result must therefore retain its logical visit count. No cache may change whether a proof succeeds or fails, including when gas is insufficient.

## Rationale

### Batched input

A single-slot API repeats the header, the account path and the top of the storage path for each slot. With the batched input, each distinct node can be sent and hashed one time, while traversal is charged for every lookup. On mainnet, 16 USDC slots need 107 unique nodes against 267 nodes for 16 separate proofs. Deduplication saves the input charge for repeated nodes; each storage lookup still pays for its full path.

The saving grows under EIP-8297. Account fields and slots 0 to 63 share one stem, and slots in the same group of 256 share a stem too. A batch of related slots then shares almost its whole path.

### Unordered node pool

The node pool has no order and no structure. It has the same shape as the execution witness that clients build for stateless validation, so clients can use the same code to build and check it. A caller can merge several `eth_getProof` responses by removing duplicate nodes.

### Header in the input, not a state root

The precompile takes the header and checks it against the block hash, so a contract does not have to parse the header, whose format changes with forks. Item 3 (`stateRoot`) and item 11 (`timestamp`) have the same position in all header versions, and EIP-8347 keeps them in place.

A separate pure "verify against a state root" call is more general. For example, it can check a rollup output root that uses the same tree. But it requires contracts to link two calls correctly. This EIP specifies only the combined call. A later EIP can add a call with a state root as input if there is demand.

### Pure precompile

The precompile does not read the state, so it does not check that `block_hash` is canonical. The caller checks this with `BLOCKHASH` or with the history contract of [EIP-2935](./eip-2935.md). A pure precompile is easy to test, has no warm or cold access cost, and gives the same result for a fixed input and executing fork.

### Tree selection by timestamp

The header timestamp defines which tree produced its state root. Contracts that use this precompile do not change when the state tree changes. Clients keep MPT verification after the migration, which is small code that they already need to serve proofs for history.

The executing fork limits which tree rules are available. A call before the migration can supply a synthetic MPT header dated far in the future. Replaying that call after the migration must still use the earlier MPT-only version, even though a new call with the same input may now select the binary tree.

The rule depends on EIP-8347, where every header commits to the whole state with one tree. Under an overlay conversion, part of the state is only in a frozen MPT whose root no header carries. A pure precompile cannot take that root from the caller safely, and the overlay rules for deletions are not specified. So this EIP fails in a conversion window instead of guessing.

### No storage root in the output

The output has only values that every planned tree supports. The EIP-8297 tree has no storage root per account, so the output does not include one.

### Failure consumes all gas

This is the same behavior as other recent precompiles. A malformed or incomplete proof is a caller error, and the caller can check the proof off-chain before it sends the transaction.

### Gas calibration

The gas has one term for each kind of work, so that the cost follows the work for every shape of input:

- `GAS_PER_NODE_WORD` pays for Keccak over the header and the nodes. It is the same as the per-word cost of the `KECCAK256` opcode.
- `GAS_PER_NODE` pays for the insertion of a node into the hash map.
- `GAS_PER_QUERY` pays for account key derivation and decoding of the account fields.
- `GAS_PER_SLOT` pays for storage key derivation and decoding of the slot value.
- `GAS_PER_TRIE_NODE` pays for resolving and examining a node during a lookup, including repeated visits to shared nodes.

Initial measurements of the Nethermind reference implementation (BenchmarkDotNet, .NET 10, one desktop x64 core) on the original test vectors gave a least-squares fit of 54 ns per word, 120 ns per node, 780 ns per query and 410 ns per slot. Those measurements did not isolate traversal cost and do not calibrate `GAS_PER_TRIE_NODE`. On the same machine, the Keccak precompile ran at 99 MGas/s for a 32-byte input and at 130 MGas/s for a 136-byte input.

The caller can construct synthetic headers and trie paths without grinding address or slot preimages. For example, a storage proof can contain 64 branch nodes followed by a leaf, and repeated slot queries traverse that path repeatedly. The traversal charge grows with that work even when the node pool is shared. Caching decoded nodes or repeated lookup results can reduce CPU work at the cost of memory and cache lookups; the gas remains independent of those optimizations.

All constants remain provisional. Before the EIP can move to Review, clients need to benchmark maximum-depth synthetic account and storage paths, repeated and distinct queries, embedded nodes, many small pool nodes, and repeated calls reusing EVM memory. Measurements must cover both tree formats and include allocation and peak-memory costs. Calldata charges do not protect against repeated internal calls. These benchmarks must confirm or revise the constants against the Keccak baseline; no throughput claim is made for the revised schedule yet.

### Related work

- [EIP-7545](./eip-7545.md) (Stagnant) proposes a pure, versioned proof verification precompile for MPT and Verkle multiproofs. It returns only success or failure, takes a state root instead of a header, and lets the caller choose the version. This EIP returns the proven values, binds them to a block hash, and selects the tree from the header.
- [RIP-7728](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7728.md) (L1SLOAD) lets an L2 read L1 storage through its sequencer's view of the latest L1 block. It reads state and verifies no proof.
- [EIP-8272](./eip-8272.md) stores roots that applications write. It does not verify state proofs, and it can store roots that this precompile then uses through a later state-root variant.

## Backwards Compatibility

This EIP adds a new precompile. It does not change existing behavior.

## Test Cases

The test vectors are in [`test-vectors.json`](../assets/eip-TBD/test-vectors.json). Each vector has `Name`, `Input`, `Expected` (hex), and `Gas` for a successful call or `ExpectedError` for a failure. `Gas` includes both input and traversal charges. Optional `AvailableGas` sets the gas supplied to the call; when omitted, sufficient gas is supplied. An optional `Context` sets `ExecutionTimestamp` and `PbtActivationTimestamp` (a timestamp or `null` for an unscheduled fork). Without `Context`, execution uses the MPT-only version with no PBT activation scheduled. Context timestamps are test-chain configuration, not precompile input. Error strings describe the expected failure; the precompile itself returns no error data. The vectors cover:

- many accounts with many slots, including an absent account, an absent slot, and an account with the empty storage root;
- an account without slots, the same account in two queries, and no queries;
- duplicate and unused nodes in the pool;
- embedded storage leaves, from two slots whose hashed keys share 9 leading nibbles;
- a missing node, a tampered node, a header that does not hash to `block_hash`, a header that is not an RLP list, trailing bytes, and truncated input;
- real mainnet headers with `eth_getProof` data for USDC (block 26268372) and WETH (block 26268373), with 1 slot and with 16 slots;
- canonical zero and maximum timestamps, leading-zero timestamps and timestamp overflow;
- historical MPT execution with a future activation configured, and tree selection immediately before, at and after activation;
- synthetic 64-branch storage paths, repeated slot queries, and exact-gas and insufficient-gas calls.

The expected outputs of the synthetic cases come from the state model that built the trie, and those of the mainnet cases come from the `eth_getProof` values. Empty binary-tree vectors exercise fork selection without depending on a hash function. Non-empty binary-tree vectors are TBD, until EIP-8297 fixes its hash function.

## Reference Implementation

An initial implementation of the MPT rules is in Nethermind, on the branch `feature/state-proof-precompile`: `StateProofPrecompile.cs`, with the tests and the vector generator in `StateProofPrecompileTests.cs`. It predates the traversal gas charge, fork-version selection and explicit timestamp validation specified here and needs updating before it can serve as a reference for this revision.

## Security Considerations

- **Canonical block.** The precompile proves only that a value is in the state of the block with hash `block_hash`. It does not prove that the block is canonical. A caller that does not check `block_hash` against `BLOCKHASH` or EIP-2935 accepts state from any header, including one that the caller made. Documentation and libraries for this precompile must show this check.
- **Denial of service.** Input charges cover header and pool processing, and each logical trie node visit is charged before it is performed. Synthetic headers can commit to maximum-depth paths without hash grinding, so fixed per-query or per-slot charges alone do not bound traversal work. Internal calls can reuse input in EVM memory without paying transaction calldata charges again; safety depends on the precompile's own gas schedule and adversarial benchmarks.
- **Absence proofs.** An implementation must accept a proof of absence only if the path ends in a node that excludes the key. An implementation that returns zero when it does not find a node gives false absence results. The rule "fail if a needed node is not in the pool" prevents this.
- **Reorgs.** A value proven against a recent block can change if that block is removed by a reorg. Callers that need final values must use finalized blocks.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
