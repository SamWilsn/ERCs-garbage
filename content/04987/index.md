---
eip: 4987
title: Held token interface
description: Interface to query ownership and balance of held tokens
author: Devin Conley (@devinaconley)
discussions-to: https://ethereum-magicians.org/t/eip-4987-held-token-standard-nfts-defi/7117
status: Stagnant
type: Standards Track
category: ERC
created: 2021-09-21
requires: 20, 165, 721, 1155
---

## Abstract

The proposed standard defines a lightweight interface to expose functional ownership and balances of held tokens. A held token is a token owned by a contract. This standard may be implemented by smart contracts which hold [EIP-20](../00020.md), [EIP-721](../00721.md), or [EIP-1155](../01155.md) tokens and is intended to be consumed by both on-chain and off-chain systems that rely on ownership and balance verification.

## Motivation

As different areas of crypto (DeFi, NFTs, etc.) converge and composability improves, there will more commonly be a distinction between the actual owner (likely a contract) and the functional owner (likely a user) of a token. Currently, this results in a conflict between mechanisms that require token deposits and systems that rely on those tokens for ownership or balance verification.

This proposal aims to address that conflict by providing a standard interface for token holders to expose ownership and balance information. This will allow users to participate in these DeFi mechanisms without giving up existing token utility. Overall, this would greatly increase interoperability across systems, benefiting both users and protocol developers.

Example implementers of this ERC standard include

- staking or farming contracts
- lending pools
- time lock or vesting vaults
- fractionalized NFT contracts
- smart contract wallets

Example consumers of this ERC standard include

- governance systems
- gaming
- PFP verification
- art galleries or showcases
- token based membership programs

## Specification

Smart contracts implementing the `ERC20` held token standard MUST implement all of the functions in the `IERC20Holder` interface.

Smart contracts implementing the `ERC20` held token standard MUST also implement `ERC165` and return true when the interface ID `0x74c89d54` is passed.

```solidity
/**
 * @notice the ERC20 holder standard provides a common interface to query
 * token balance information
 */
interface IERC20Holder is IERC165 {
  /**
   * @notice emitted when the token is transferred to the contract
   * @param owner functional token owner
   * @param tokenAddress held token address
   * @param tokenAmount held token amount
   */
  event Hold(
    address indexed owner,
    address indexed tokenAddress,
    uint256 tokenAmount
  );

  /**
   * @notice emitted when the token is released back to the user
   * @param owner functional token owner
   * @param tokenAddress held token address
   * @param tokenAmount held token amount
   */
  event Release(
    address indexed owner,
    address indexed tokenAddress,
    uint256 tokenAmount
  );

  /**
   * @notice get the held balance of the token owner
   * @dev should throw for invalid queries and return zero for no balance
   * @param tokenAddress held token address
   * @param owner functional token owner
   * @return held token balance
   */
  function heldBalanceOf(address tokenAddress, address owner)
    external
    view
    returns (uint256);
}

```

Smart contracts implementing the `ERC721` held token standard MUST implement all of the functions in the `IERC721Holder` interface.

Smart contracts implementing the `ERC721` held token standard MUST also implement `ERC165` and return true when the interface ID `0x16b900ff` is passed.

```solidity
/**
 * @notice the ERC721 holder standard provides a common interface to query
 * token ownership and balance information
 */
interface IERC721Holder is IERC165 {
  /**
   * @notice emitted when the token is transferred to the contract
   * @param owner functional token owner
   * @param tokenAddress held token address
   * @param tokenId held token ID
   */
  event Hold(
    address indexed owner,
    address indexed tokenAddress,
    uint256 indexed tokenId
  );

  /**
   * @notice emitted when the token is released back to the user
   * @param owner functional token owner
   * @param tokenAddress held token address
   * @param tokenId held token ID
   */
  event Release(
    address indexed owner,
    address indexed tokenAddress,
    uint256 indexed tokenId
  );

  /**
   * @notice get the functional owner of a held token
   * @dev should throw for invalid queries and return zero for a token ID that is not held
   * @param tokenAddress held token address
   * @param tokenId held token ID
   * @return functional token owner
   */
  function heldOwnerOf(address tokenAddress, uint256 tokenId)
    external
    view
    returns (address);

  /**
   * @notice get the held balance of the token owner
   * @dev should throw for invalid queries and return zero for no balance
   * @param tokenAddress held token address
   * @param owner functional token owner
   * @return held token balance
   */
  function heldBalanceOf(address tokenAddress, address owner)
    external
    view
    returns (uint256);
}
```

Smart contracts implementing the `ERC1155` held token standard MUST implement all of the functions in the `IERC1155Holder` interface.

Smart contracts implementing the `ERC1155` held token standard MUST also implement `ERC165` and return true when the interface ID `0xced24c37` is passed.

```solidity
/**
 * @notice the ERC1155 holder standard provides a common interface to query
 * token balance information
 */
interface IERC1155Holder is IERC165 {
  /**
   * @notice emitted when the token is transferred to the contract
   * @param owner functional token owner
   * @param tokenAddress held token address
   * @param tokenId held token ID
   * @param tokenAmount held token amount
   */
  event Hold(
    address indexed owner,
    address indexed tokenAddress,
    uint256 indexed tokenId,
    uint256 tokenAmount
  );

  /**
   * @notice emitted when the token is released back to the user
   * @param owner functional token owner
   * @param tokenAddress held token address
   * @param tokenId held token ID
   * @param tokenAmount held token amount
   */
  event Release(
    address indexed owner,
    address indexed tokenAddress,
    uint256 indexed tokenId,
    uint256 tokenAmount
  );

  /**
   * @notice get the held balance of the token owner
   * @dev should throw for invalid queries and return zero for no balance
   * @param tokenAddress held token address
   * @param owner functional token owner
   * @param tokenId held token ID
   * @return held token balance
   */
  function heldBalanceOf(
    address tokenAddress,
    address owner,
    uint256 tokenId
  ) external view returns (uint256);
}
```

## Rationale

This interface is designed to be extremely lightweight and compatible with any existing token contract. Any token holder contract likely already stores all relevant information, so this standard is purely adding a common interface to expose that data.

The token address parameter is included to support contracts that can hold multiple token contracts simultaneously. While some contracts may only hold a single token address, this is more general to either scenario.

Separate interfaces are proposed for each token type (EIP-20, EIP-721, EIP-1155) because any contract logic to support holding these different tokens is likely independent. In the scenario where a single contract does hold multiple token types, it can simply implement each appropriate held token interface.


## Backwards Compatibility

Importantly, the proposed specification is fully compatible with all existing EIP-20, EIP-721, and EIP-1155 token contracts.

Token holder contracts will need to be updated to implement this lightweight interface.

Consumer of this standard will need to be updated to respect this interface in any relevant ownership logic.


## Reference Implementation

A full example implementation including [interfaces](./assets/IERC721Holder.sol), a vault [token holder](./assets/Vault.sol), and a [consumer](./assets/Consumer.sol), can be found at `assets/eip-4987/`.

Notably, consumers of the `IERC721Holder` interface can do a chained lookup for the owner of any specific token ID using the following logic.

```solidity
  /**
   * @notice get the functional owner of a token
   * @param tokenId token id of interest
   */
  function getOwner(uint256 tokenId) external view returns (address) {
    // get raw owner
    address owner = token.ownerOf(tokenId);

    // if owner is not contract, return
    if (!owner.isContract()) {
      return owner;
    }

    // check for token holder interface support
    try IERC165(owner).supportsInterface(0x16b900ff) returns (bool ret) {
      if (!ret) return owner;
    } catch {
      return owner;
    }

    // check for held owner
    try IERC721Holder(owner).heldOwnerOf(address(token), tokenId) returns (address user) {
      if (user != address(0)) return user;
    } catch {}

    return owner;
  }
```


## Security Considerations

Consumers of this standard should be cautious when using ownership information from unknown contracts. A bad actor could implement the interface, but report invalid or malicious information with the goal of manipulating a governance system, game, membership program, etc.

Consumers should also verify the overall token balance and ownership of the holder contract as a sanity check.


## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
