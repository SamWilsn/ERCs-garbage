---
eip: 7590
title: ERC-20 Holder Extension for NFTs
description: Extension to allow NFTs to receive and transfer ERC-20 tokens.
author: Steven Pineda (@steven2308), Jan Turk (@ThunderDeliverer)
discussions-to: https://ethereum-magicians.org/t/token-holder-extension-for-nfts/16260
status: Review
type: Standards Track
category: ERC
created: 2024-01-05
requires: 20, 165, 721
---

## Abstract

This proposal suggests an extension to [ERC-721](../00721.md) to enable easy exchange of [ERC-20](../00020.md) tokens. By enhancing [ERC-721](../00721.md), it allows unique tokens to manage and trade [ERC-20](../00020.md) fungible tokens bundled in a single NFT. This is achieved by including methods to pull [ERC-20](../00020.md) tokens into the NFT contract to a specific NFT, and transferring them out by the owner of such NFT. A transfer out nonce is included to prevent front-running issues.

## Motivation

In the ever-evolving landscape of blockchain technology and decentralized ecosystems, interoperability between diverse token standards has become a paramount concern. By enhancing [ERC-721](../00721.md) functionality, this proposal empowers non-fungible tokens (NFTs) to engage in complex transactions, facilitating the exchange of fungible tokens, unique assets, and multi-class assets within a single protocol.

This ERC introduces new utilities in the following areas:
- Expanded use cases
- Facilitating composite transactions
- Market liquidity and value creation

### Expanded Use Cases

Enabling [ERC-721](../00721.md) tokens to handle various token types opens the door to a wide array of innovative use cases. From gaming and digital collectibles to decentralized finance (DeFi) and supply chain management, this extension enhances the potential of NFTs by allowing them to participate in complex, multi-token transactions.

### Facilitating Composite Transactions

With this extension, composite transactions involving both fungible and non-fungible assets become easier. This functionality is particularly valuable for applications requiring intricate transactions, such as gaming ecosystems where in-game assets may include a combination of fungible and unique tokens.

### Market Liquidity and Value Creation

By allowing [ERC-721](../00721.md) tokens to hold and trade different types of tokens, it enhances liquidity for markets in all types of tokens.

## Specification

```solidity

interface IERC7590 /*is IERC165, IERC721*/  {
    /**
     * @notice Used to notify listeners that the token received ERC-20 tokens.
     * @param erc20Contract The address of the ERC-20 smart contract
     * @param toTokenId The ID of the token receiving the ERC-20 tokens
     * @param from The address of the account from which the tokens are being transferred
     * @param amount The number of ERC-20 tokens received
     */
    event ReceivedERC20(
        address indexed erc20Contract,
        uint256 indexed toTokenId,
        address indexed from,
        uint256 amount
    );

    /**
     * @notice Used to notify the listeners that the ERC-20 tokens have been transferred.
     * @param erc20Contract The address of the ERC-20 smart contract
     * @param fromTokenId The ID of the token from which the ERC-20 tokens have been transferred
     * @param to The address receiving the ERC-20 tokens
     * @param amount The number of ERC-20 tokens transferred
     */
    event TransferredERC20(
        address indexed erc20Contract,
        uint256 indexed fromTokenId,
        address indexed to,
        uint256 amount
    );

    /**
     * @notice Used to retrieve the given token's specific ERC-20 balance
     * @param erc20Contract The address of the ERC-20 smart contract
     * @param tokenId The ID of the token being checked for ERC-20 balance
     * @return The amount of the specified ERC-20 tokens owned by a given token
     */
    function balanceOfERC20(
        address erc20Contract,
        uint256 tokenId
    ) external view returns (uint256);

    /**
     * @notice Transfer ERC-20 tokens from a specific token.
     * @dev The balance MUST be transferred from this smart contract.
     * @dev MUST increase the transfer-out-nonce for the tokenId
     * @dev MUST revert if the `msg.sender` is not the owner of the NFT or approved to manage it.
     * @param erc20Contract The address of the ERC-20 smart contract
     * @param tokenId The ID of the token to transfer the ERC-20 tokens from
     * @param amount The number of ERC-20 tokens to transfer
     * @param data Additional data with no specified format, to allow for custom logic
     */
    function transferHeldERC20FromToken(
        address erc20Contract,
        uint256 tokenId,
        address to,
        uint256 amount,
        bytes memory data
    ) external;

    /**
     * @notice Transfer ERC-20 tokens to a specific token.
     * @dev The ERC-20 smart contract must have approval for this contract to transfer the ERC-20 tokens.
     * @dev The balance MUST be transferred from the `msg.sender`.
     * @param erc20Contract The address of the ERC-20 smart contract
     * @param tokenId The ID of the token to transfer ERC-20 tokens to
     * @param amount The number of ERC-20 tokens to transfer
     * @param data Additional data with no specified format, to allow for custom logic
     */
    function transferERC20ToToken(
        address erc20Contract,
        uint256 tokenId,
        uint256 amount,
        bytes memory data
    ) external;

    /**
     * @notice Nonce increased every time an ERC20 token is transferred out of a token
     * @param tokenId The ID of the token to check the nonce for
     * @return The nonce of the token
     */
    function erc20TransferOutNonce(
        uint256 tokenId
    ) external view returns (uint256);
}
```


## Rationale

### Pull Mechanism

We propose using a pull mechanism, where the contract transfers the token to itself, instead of receiving it via "safe transfer" for 2 reasons:

1. Customizability with Hooks. By initiating the process this way, smart contract developers have the flexibility to execute specific actions before and after transferring the tokens.

2. Lack of transfer with callback: [ERC-20](../00020.md) tokens lack a standardized transfer with callback method, such as the "safeTransfer" on [ERC-721](../00721.md), which means there is no reliable way to notify the receiver of a successful transfer, nor to know which is the destination token is.

This has the disadvantage of requiring approval of the token to be transferred before actually transferring it into an NFT.

### Granular vs Generic

We considered 2 ways of presenting the proposal:
1. A granular approach where there is an independent interface for each type of held token.
2. A universal token holder which could also hold and transfer [ERC-721](../00721.md) and [ERC-1155](../01155.md).

An implementation of the granular version is slightly cheaper in gas, and if you're using just one or two types, it's smaller in contract size. The generic version is smaller and has single methods to send or receive, but it also adds some complexity by always requiring Id and amount on transfer methods. Id not being necessary for [ERC-20](../00020.md) and amount not being necessary for [ERC-721](../00721.md).

We also considered that due to the existence of safe transfer methods on both [ERC-721](../00721.md) and [ERC-1155](../01155.md), and the commonly used interfaces of `IERC721Receiver` and `IERC1155Receiver`, there is not much need to declare an additional interface to manage such tokens. However, this is not the case for [ERC-20](../00020.md), which does not include a method with a callback to notify the receiver of the transfer.

For the aforementioned reasons, we decided to go with a granular approach.


## Backwards Compatibility

No backward compatibility issues found.

## Test Cases

Tests are included in [`erc7590.ts`](./assets/test/erc7590.ts).

To run them in terminal, you can use the following commands:

```
cd ../assets/eip-erc7590
npm install
npx hardhat test
```

## Reference Implementation

See [`ERC7590Mock.sol`](./assets/contracts/ERC7590Mock.sol).

## Security Considerations

The same security considerations as with [ERC-721](../00721.md) apply: hidden logic may be present in any of the functions, including burn, add resource, accept resource, and more.

Caution is advised when dealing with non-audited contracts.

Implementations MUST use the message sender as from parameter when they are transferring tokens into an NFT. Otherwise, since the current contract needs approval, it could potentially pull the external tokens into a different NFT.

When transferring [ERC-20](../00020.md) tokens in or out of an NFT, it could be the case that the amount transferred is not the same as the amount requested. This could happen if the [ERC-20](../00020.md) contract has a fee on transfer. This could cause a bug on your Token Holder contract if you do not manage it properly. There are 2 ways to do it, both of which are valid:
1. Use the `IERC20` interface to check the balance of the contract before and after the transfer, and revert if the balance is not the expected one, hence not supporting tokens with fees on transfer.
2. Use the `IERC20` interface to check the balance of the contract before and after the transfer, and use the difference to calculate the amount of tokens that were actually transferred. 

To prevent a seller from front running the sale of an NFT holding [ERC-20](../00020.md) tokens to transfer out such tokens before a sale is executed, marketplaces MUST beware of the `erc20TransferOutNonce` and revert if it has changed since listed.

[ERC-20](../00020.md) tokens that are transferred directly to the NFT contract will be lost.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
