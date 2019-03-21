---
eip: TODO
title: Add `eth_getStateDiff` method to JSON-RPC Interface
author: Fabio Berger (@fabioberger)
discussions-to: <URL>
status: Draft
type: Interface
category: Interface
created: 2019-03-20
---

<!--You can leave these HTML comments in your merged EIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new EIPs. Note that an EIP number will be assigned by an editor. When opening a pull request to submit your EIP, please use an abbreviated title in the filename, `eip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary

<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the EIP.-->

Add a JSON-RPC method that returns the [state trie](https://github.com/ethereum/wiki/wiki/Patricia-Tree#state-trie) diffs introduced between any two blocks.

## Abstract

<!--A short (~200 word) description of the technical issue being addressed.-->

There is currently no way for developers to retrieve the state trie or storage trie changes introduced between two blocks. Trie diffs are a very efficient way to figure out what blockchain state has changed between blocks, _without_ re-executing and tracing every transactions within the blocks.

## Motivation

<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

#### Ethereum balance watching

One example use-case for requesting a state trie diff is efficiently tracking an arbitrarily large set of Ethereum balances. ETH transfers do not emit events and so there are currently only two ways to update ETH balances when a new block is mined: re-fetch all ETH balances for all addresses of interest or request traces for every transaction within the block, and re-fetch balances for all addresses involved. The former is simple but neither way is efficient.

With `eth_getStateDiff`, a single JSON-RPC request will return all ETH balances modified by all transactions within a block range. Simple and efficient.

#### Arbitrary state watching

The only available way to watch for contract state updates today is for a contract to emit an event whenever a piece of state is modified. Examples include the `Transfer` event in the ERC20 standard, which _SHOULD_ be triggered whenever a transfer is done. Conceivably one could parse the logs for all transactions in a block and know which user balances have changed. Unfortunately this is still not the case. Not all ERC20's implement the specification properly, some omitting the `Transfer` event. Even tokens that do follow the specification to the letter have no guidance on whether to fire `Transfer` events when minting or burning tokens. Rather then rely on an event, a proxy for the state change, `eth_getStateDiff` would allow anyone to watch the state itself.

As proposed by @recmo in [this RFC](https://github.com/ethereum/EIPs/issues/781), an alternative way of implementing state watching would be to:

1. execute any view function (e.g `balanceOf()`) as if it was a STATICCALL (see limitations below).
2. Trace and remember all the storage locations accessed with SLOAD.

When a new block is mined, call `eth_getStateDiff` for the block and see if any of the storage locations of interest have been modified. If so, re-compute the balance for the associated address. No other balances need be re-computed. If the application goes offline for some time, it can recover by requesting a state diff between it's last known block and the latest block.

This approach is flexible enough to allow developers to efficiently watch any arbitrary contract state. It also allows developers to build applications that are more robust and cannot be easily tricked by incorrect or malicious implementations of contracts that omit events while modifying state. This is an especially important consideration for financial applications.

**Limitations**: The semantics of the function call to monitor are similar to that of STATICCALL, with the additional restriction that non-deterministic opcodes (BLOCKHASH, COINBASE, …) should not be used (unless the application has other means of handling false negatives).

## Specification

<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

### eth_getStateDiff

Returns the state trie and nested storage trie diffs of the block range.

```
{
    "method": "eth_getStateDiff",
    "params": [
        "number",
        "0x2ed119",
        "0x2ed11e"
    ],
    "id": 1,
    "jsonrpc": "2.0"
}
```

Parameters

1. The block identifier type (either "hash" or "number")
2. From block identifier (either a block hash or number)
3. To block identifier (either a block hash or number) [Optional -- if omitted, gets diff for single block]

Returns

`Array` - An array of state diffs:

- `address`: DATA|Array, 20 Bytes - Externally-owned or Contract address
- `nonce`: Nonce diff
- `balance`: Ether balance diff
- `storageDiff`: Storage diff. An object where keys are the storage locations modified and the values contain the diff.
- `code`: Code diff

```
{
    "result": [
    {
        "address": "0x5409ed021d9299bf6814279a6a1411a7e866a631",
        "nonce": {
            "from": "0x69",
            "to": "0x6a"
        },
        "balance": {
            "from": "0x206410d597b1d955c79",
            "to": "0x206405b0a6d50037619"
        },
        "storageDiff": {
            "0x0000000000000000000000000000000000000000000000000000000000000005": {
                "*": {
                    "from": "0x00000000000000000000000000000000000000000000014a20172df63365805a",
                    "to": "0x00000000000000000000000000000000000000000000014a201e48f37cf2805a"
                }
            },
            ...
        },
        "code": "=",
    },
    { ... }
    ],
    "id": 1,
    "jsonrpc": "2.0"
}
```

If the diff is empty, we simply output `=`. If the diff is non-empty, an object is returned with two properties (`from` - the previous value, and `to` - the new value).

## Rationale

<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

The endpoints design attempts to remain as simple, and use-case agnostic as possible. It simply returns the state diff using the already defined [state trie](https://github.com/ethereum/wiki/wiki/Patricia-Tree#state-trie) contents. Instead of returning the diff of the storage root hash, it goes one level deeper and also diffs the associated storage tries.

### Performance

The state and storage tries are [merkle particia tries](https://github.com/ethereum/wiki/wiki/Patricia-Tree#main-specification-merkle-patricia-trie) and as such, have some nice performance characteristics. When diffing two tries, a traversal can ignore entire sub-tries for which parent nodes are identical. For the remaining nodes, a max traversal of `2 * 256` nodes is required to reach a leaf value of the trie (in reality it will be less thanks to [extension nodes](https://github.com/ethereum/wiki/wiki/Patricia-Tree#optimization)). This cost is also amortized between multiple modified leafs that are close to one another, as is the case for contract's storage. Beyond this constant cost, the remaining cost scales linearly with the number of differences between the two tries.

Since there can be many state differences introduced within a sufficiently large block range, Ethereum clients should implement a request timeout that if reached, aborts the request. Despite having a request timeout on these requests, fetching state diffs for block ranges will be much more efficient then fetching every block.

### Related work

The Geth team has already implemented a custom [`debug_getModifiedAccountsByNumber`](https://github.com/ethereum/go-ethereum/blob/91eec1251c06727581063cd7e942ba913d806971/eth/api.go#L421) JSON-RPC method that returns the addresses with modified ETH balances for a given block range. They have identified the need for a more efficient way of performing this task.

The Parity team has implemented a custom [`trace_replayBlockTransactions`](https://wiki.parity.io/JSONRPC-trace-module#trace_replayblocktransactions) JSON-RPC method that returns the ETH balance diffs as well as storage trie diffs for all transactions within a block. They too have identified the need to expose this functionality to their users.

This EIP is largely an attempt to standardize these two approaches such that projects that wish to build solutions that aren't beholden to any single Ethereum client implementation may do so with confidence. Neither of the above methods are part of the standard and could be modified or removed easily.

## Backwards Compatibility

<!--All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

This EIP is fully backward compatible.

## Implementation

<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

TODO

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

```

```
