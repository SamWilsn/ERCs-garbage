---
eip: 7410
title: ERC-20 Update Allowance By Spender
description: Extension to enable revoking and decreasing allowance approval by spender for ERC-20
author: Mohammad Zakeri Rad (@zakrad), Adam Boudjemaa (@aboudjem), Mohamad Hammoud (@mohamadhammoud)
discussions-to: https://ethereum-magicians.org/t/eip-7410-decrease-allowance-by-spender/15222
status: Draft
type: Standards Track
category: ERC
created: 2023-07-26
requires: 20, 165
---

## Abstract

This extension adds a `decreaseAllowanceBySpender` function to decrease [ERC-20](../00020.md) allowances, in which a spender can revoke or decrease a given allowance by a specific address. This ERC extends [ERC-20](../00020.md).

## Motivation

Currently, [ERC-20](../00020.md) tokens offer allowances, enabling token owners to authorize spenders to use a designated amount of tokens on their behalf. However, the process of decreasing an allowance is limited to the owner's side, which can be problematic if the token owner is a treasury wallet or a multi-signature wallet that has granted an excessive allowance to a spender. In such cases, reducing the allowance from the owner's perspective can be time-consuming and challenging.

To address this issue and enhance security measures, this ERC proposes allowing spenders to decrease or revoke the granted allowance from their end. This feature provides an additional layer of security in the event of a potential hack in the future. It also eliminates the need for a consensus or complex procedures to decrease the allowance from the token owner's side.

## Specification

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

Contracts using this ERC MUST implement the `IERC7410` interface.

### Interface implementation

```solidity
pragma solidity ^0.8.0;

/**
 * @title IERC-7410 Update Allowance By Spender Extension
 * Note: the ERC-165 identifier for this interface is 0x12860fba
 */
interface IERC7410 is IERC20 {

    /**
     * @notice Decreases any allowance by `owner` address for caller.
     * Emits an {IERC20-Approval} event.
     *
     * Requirements:
     * - when `subtractedValue` is equal or higher than current allowance of spender the new allowance is set to 0.
     * Nullification also MUST be reflected for current allowance being type(uint256).max.
     */
    function decreaseAllowanceBySpender(address owner, uint256 subtractedValue) external;

}
```

The `decreaseAllowanceBySpender(address owner, uint256 subtractedValue)` function MUST be either `public` or `external`.

The `Approval` event MUST be emitted when `decreaseAllowanceBySpender` is called.

The `supportsInterface` method MUST return `true` when called with `0x12860fba`.

## Rationale

The technical design choices within this ERC are driven by the following considerations:

- The introduction of the `decreaseAllowanceBySpender` function empowers spenders by allowing them to autonomously revoke or decrease allowances. This design choice aligns with the goal of providing more direct control to spenders over their authorization levels.
- The requirement for the `subtractedValue` to be lower than the current allowance ensures a secure implementation. Additionally, nullification is achieved by setting the new allowance to 0 when `subtractedValue` is equal to or exceeds the current allowance. This approach adds an extra layer of security and simplifies the process of decreasing allowances.
- The decision to maintain naming patterns similar to [ERC-20](../00020.md)'s approvals is rooted in promoting consistency and ease of understanding for developers familiar with [ERC-20](../00020.md) standard.

## Backwards Compatibility

This standard is compatible with [ERC-20](../00020.md).

## Reference Implementation

An minimal implementation is included [here](./assets/ERC7410.sol).

## Security Considerations

Users of this ERC must thoroughly consider the amount of tokens they decrease from their allowance for an `owner`.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
