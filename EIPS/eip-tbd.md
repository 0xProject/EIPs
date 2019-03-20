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

Add a JSON-RPC method that returns the [state trie](https://github.com/ethereum/wiki/wiki/Patricia-Tree#state-trie) differences introduced by executing the transactions within a specific block.

## Abstract

<!--A short (~200 word) description of the technical issue being addressed.-->

There is currently no way for developers to retrieve the state trie or storage trie changes introduced by the transactions in a block. Trie diffs are a very efficient way to quickly figure out what blockchain state has changed, without re-executing every transactions within a block.

## Motivation

<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

#### Ethereum balance watching

One example use-case for requesting a state trie diff is efficiently tracking an arbitrarily large set of Ethereum balances. ETH transfers do not emit events and so there are currently only two ways to re-compute ETH balances when a new block is mined: re-fetch all ETH balances for all addresses of interest or request traces for every transaction within the block, find all calls to `Address(X).transfer`, see what address it was called with and re-fetch balances for that set of addresses. The former is simple but neither way is efficient.

With `eth_getStateDiff`, a single JSON-RPC request will return all ETH balances modified by the block's transactions. Simple and efficient.

#### Arbitrary state watching

The only available way to watch for contract state updates today is for a contract to emit an event whenever a piece of state is modified. Examples include the `Transfer` event in the ERC20 standard, which _SHOULD_ be triggered whenever a transfer is done. Conceivably one could parse the logs for all transactions in a block and know which user balances have changed. Unfortunately this is still not the case. Not all ERC20's implement the specification properly, and some omit the `Transfer` event. Even tokens that do follow the specification to the letter have no guidance on whether to fire `Transfer` events when minting or burning tokens. Rather then rely on an event, a proxy for the state change, `eth_getStateDiff` would allow anyone to watch the state itself.

As proposed by @recmo in [this RFC](https://github.com/ethereum/EIPs/issues/781), an alternative way of implementing state watching would be to:

1. execute any view function (e.g `balanceOf()`) as if it was a STATICCALL (see limitations below).
2. Trace and remember all the storage locations accessed with SLOAD.

When a new block is mined, call `eth_getStateDiff` for the block and see if any of the storage locations of interest have been modified. If so, re-compute the balance for the associated user. All other user balances do not need to be re-computed.

This approach does not rely on developers adhering to ERC standards and can be used to watch arbitrary state, whether or not the developer of a contract thought it important enough to emit an event for it.

## Specification

<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

### eth_getStateDiff

Returns the state trie and storage trie diffs for the requested block.

````
{
    "method": "eth_getStateDiff",
    "params": [
        "0x2ed119",
        "number"
    ],
    "id": 1,
    "jsonrpc": "2.0"
}
```

Parameters

1. Block identifier (either a block hash or number)
2. The block identifier type (either "hash" or "number")


Returns

`Array` - An array of state diffs:

-   `address`: DATA|Array, 20 Bytes - Externally-owned or Contract address
-   `nonce`: Associated Nonce diff
-   `balance`: Associated Ether balance diff
-   `storageDiff`: Associated storage diff. An object where keys are the storage locations modified and the values contain the diff.
-   `code`: Associated code diff

```
{
  "id": 1,
  "jsonrpc": "2.0",
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
  ]
}
```

If the diff is empty, we simply output `=`. If the diff is non-empty, an object is returned with two properties (`from` - the previous value, and `to` - the new value).


## Rationale

<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

The endpoints design attempts to remain as simple, and use-case agnostic as possible. It simply returns the state diff using the already defined [state trie](https://github.com/ethereum/wiki/wiki/Patricia-Tree#state-trie) contents. Instead of simple a diff of the storage root, it goes one level deeper and also diffs the associate storage tries.

### Related work

The Geth team has already implemented a custom [`debug_getModifiedAccountsByNumber`](https://github.com/ethereum/go-ethereum/blob/91eec1251c06727581063cd7e942ba913d806971/eth/api.go#L421) JSON-RPC method that returns the addresses with modified ETH balances for a given block. They have identified the need for a more efficient way of performing this task.

The Parity team has implemented a custom [`trace_replayBlockTransactions`](https://wiki.parity.io/JSONRPC-trace-module#trace_replayblocktransactions) JSON-RPC method that returns the ETH balance diffs as well as storage trie diffs for all transactions within a block. They too have identified the need to expose functionality to their users.

The EIP is largely an attempt to standardize these two approaches such that projects that wish to build solutions that aren't beholden to any single Ethereum client implementation may do so with confidence. Neither of the above methods are part of the standard and could be modified or removed easily.

## Backwards Compatibility

<!--All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

This EIP is fully backward compatible.

## Implementation

<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

TODO

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
````
