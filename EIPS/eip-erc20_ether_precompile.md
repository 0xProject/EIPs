---
eip: <to be assigned>
title: EIP20 Ether precompile
author: Remco Bloemen <remco@0xproject.com>
discussions-to: https://ethereum-magicians.org/t/<to be assigned>
status: Draft
type: Standards Track
category: Core
created: 2018-07-18
requires: 20
---

<!--You can leave these HTML comments in your merged EIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new EIPs. Note that an EIP number will be assigned by an editor. When opening a pull request to submit your EIP, please use an abbreviated title in the filename, `eip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the EIP.-->

A precompile that provides an [EIP-20][eip20] interface for native Ether.

[eip20]: https://eips.ethereum.org/EIPS/eip-20

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->

* Adds allowance mechanism to Ether.
* Allows querying the total supply of Ether.
* Allows Ether transfers without a call (i.e. no fallback gets triggered).


## Motivation
<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

Currently DApps that support tokens can not work with Ether without a workaround such as [WETH][weth]. WEth introduces extra steps and confusion for the end user. It requires deposit and withdrawal. It generates two separate Ether balances for the same address.

[weth]: https://weth.io/

Alternatively, a DApp could try to special case the handling of Ether. This creates extra complexity in the smart contracts, leading to errors. Ether does not have an allowance mechanism, which means the contracts will have to be custodial.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

If `block.number >= XXXXX`, then at `address = 0x00......06` a precompile that implements the [EIP-20][eip20] interface according to the [ABI][abi] with the following semantics attached to the functions:

This precompile has the following state:

* Ether balances. (Optional, we can also keep these at the owner address location.)
* Allowances.

[abi]: http://solidity.readthedocs.io/en/v0.4.24/abi-spec.html


```solidity
function name() view returns (string name)
```
Returns `"Ether"`.\
Gas cost: TBD

```solidity
function symbol() view returns (string symbol)
```
Returns `"ETH"`.\
Gas cost: TBD

```solidity
function decimals() view returns (uint8 decimals)
```
Returns `18`.\
Gas cost: TBD

```solidity
function totalSupply() view returns (uint256 totalSupply)
```
Returns the sum of all Ether balances in Wei. No current equivalent.\
Gas cost: TBD

```solidity
function balanceOf(address _owner) view returns (uint256 balance)
```
Returns the Ethereum balance of address `_owner` in Wei, identical to the existing `BALANCE` opcode.\
Gas cost: TBD

```solidity
function transfer(address _to, uint256 _value) returns (bool success)
```
Transfers `_value` Wei from the caller to `_to`. Unlike Ether transfer using the `CALL` instruction, it is a pure transfer of funds and no contract code at `_to` is executed. Returns `true` on success and fires a log event or reverts if the balance of the caller is less than `_value`.\
Gas cost: TBD

```solidity
function transferFrom(address _from, address _to, uint256 _value) returns (bool success)
```
Transfers `_value` Wei from the `_from` to `_to` and deducts the allowance of the caller . Unlike Ether transfer using the `CALL` instruction, it is a pure transfer of funds and no contract code at `_to` is executed. Returns `true` on success. Reverts if the allowance or balance is insufficient.\
Gas cost: TBD

```solidity
function approve(address _spender, uint256 _value) returns (bool success)
```
Sets or resets an allowance .
Fires a log event.

TBD: Implement front-running protection?

```solidity
function allowance(address _owner, address _spender) view returns (uint256 remaining)
```
Returns the current allowance of sender on `_owner`.

```solidity
event Transfer(address indexed _from, address indexed _to, uint256 _value)
```

```solidity
event Approval(address indexed _owner, address indexed _spender, uint256 _value)
```


## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

## Backwards Compatibility
<!--All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

## Test Cases
<!--Test cases for an implementation are mandatory for EIPs that are affecting consensus changes. Other EIPs can choose to include links to test cases if applicable.-->

## Implementation
<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->
The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
