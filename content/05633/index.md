---
eip: 5633
title: Composable Soulbound NFT, EIP-1155 Extension
description: Add composable soulbound property to EIP-1155 tokens
author: HonorLabs (@honorworldio)
discussions-to: https://ethereum-magicians.org/t/composable-soulbound-nft-eip-1155-extension/10773
status: Stagnant
type: Standards Track
category: ERC
created: 2022-09-09
requires: 165, 1155
---

## Abstract

This standard is an extension of [EIP-1155](../01155.md). It proposes a smart contract interface that can represent any number of soulbound and non-soulbound NFT types. Soulbound is the property of a token that prevents it from being transferred between accounts. This standard allows for each token ID to have its own soulbound property. 

## Motivation

The soulbound NFTs similar to World of Warcraft’s soulbound items are attracting more and more attention in the Ethereum community. In a real world game like World of Warcraft, there are thousands of items, and each item has its own soulbound property. For example, the amulate Necklace of Calisea is of soulbound property, but another low level amulate is not. This proposal provides a standard way to represent soulbound NFTs that can coexist with non-soulbound ones. It is easy to design a composable NFTs for an entire collection in a single contract. 

This standard outline a interface to EIP-1155 that allows wallet implementers and developers to check for soulbound property of token ID using [EIP-165](../00165.md). the soulbound property can be checked in advance, and the transfer function can be called only when the token is not soulbound.

## Specification
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

A token type with a `uint256 id`  is soulbound if function `isSoulbound(uint256 id)` returning true. In this case, all EIP-1155 functions of the contract that transfer the token from one account to another MUST throw, except for mint and burn. 

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.0;

interface IERC5633 {
  /**
   * @dev Emitted when a token type `id` is set or cancel to soulbound, according to `bounded`.
   */
  event Soulbound(uint256 indexed id, bool bounded);

  /**
   * @dev Returns true if a token type `id` is soulbound.
   */
  function isSoulbound(uint256 id) external view returns (bool);
}
```
Smart contracts implementing this standard MUST implement the EIP-165 supportsInterface function and MUST return the constant value true if 0x911ec470 is passed through the interfaceID argument.

## Rationale

If all tokens in a contract are soulbound by default, `isSoulbound(uint256 id)` should return true by default during implementation.

## Backwards Compatibility

This standard is fully EIP-1155 compatible.

## Test Cases

Test cases are included in [test.js](./assets/test/test.js). 

Run in terminal:

```bash
cd ../assets/eip-5633
npm install
npx hardhat test
```

Test contract are included in [`ERC5633Demo.sol`](./assets/contracts/ERC5633Demo.sol). 

## Reference Implementation

See [`ERC5633.sol`](./assets/contracts/ERC5633.sol).

## Security Considerations

There are no security considerations related directly to the implementation of this standard.

## Copyright
Copyright and related rights waived via [CC0](/LICENSE.md).
