---
eip: 7507
title: Multi-User NFT Extension
description: An extension of ERC-721 to allow multiple users to a token with restricted permissions.
author: Ming Jiang (@minkyn), Zheng Han (@hanbsd), Fan Yang (@fayang)
discussions-to: https://ethereum-magicians.org/t/eip-7507-multi-user-nft-extension/15660
status: Draft
type: Standards Track
category: ERC
created: 2023-08-24
requires: 721
---

## Abstract

This standard is an extension of [ERC-721](../00721.md). It proposes a new role `user` in addition to `owner` for a token. A token can have multiple users under separate expiration time. It allows the subscription model where an NFT can be subscribed non-exclusively by different users.

## Motivation

Some NFTs represent IP assets, and IP assets have the need to be licensed for access without transferring ownership. The subscription model is a very common practice for IP licensing where multiple users can subscribe to an NFT to obtain access. Each subscription is usually time limited and will thus be recorded with an expiration time.

Existing [ERC-4907](../04907.md) introduces a similar feature, but does not allow for more than one user. It is more suitable in the rental scenario where a user gains an exclusive right of use to an NFT before the next user. This rental model is common for NFTs representing physical assets like in games, but not very useful for shareable IP assets.

## Specification

Solidity interface available at [`IERC7507.sol`](./assets/contracts/IERC7507.sol):

```solidity
interface IERC7507 {

    /// @notice Emitted when the expires of a user for an NFT is changed
    event UpdateUser(uint256 indexed tokenId, address indexed user, uint64 expires);

    /// @notice Get the user expires of an NFT
    /// @param tokenId The NFT to get the user expires for
    /// @param user The user to get the expires for
    /// @return The user expires for this NFT
    function userExpires(uint256 tokenId, address user) external view returns(uint256);

    /// @notice Set the user expires of an NFT
    /// @param tokenId The NFT to set the user expires for
    /// @param user The user to set the expires for
    /// @param expires The user could use the NFT before expires in UNIX timestamp
    function setUser(uint256 tokenId, address user, uint64 expires) external;

}
```

## Rationale

This standard complements [ERC-4907](../04907.md) to support multi-user feature. Therefore the proposed interface tries to keep consistent using the same naming for functions and parameters.

However, we didn't include the corresponding `usersOf(uint256 tokenId)` function as that would imply the implemention has to support enumerability over multiple users. It is not always necessary, for example, in the case of open subscription. So we decide not to add it to the interface and leave the choice up to the implementers.

## Backwards Compatibility

No backwards compatibility issues found.

## Test Cases

Test cases available available at: [`ERC7507.test.ts`](./assets/test/ERC7507.test.ts):

```typescript
import { loadFixture } from "@nomicfoundation/hardhat-toolbox/network-helpers";
import { expect } from "chai";
import { ethers } from "hardhat";

const NAME = "NAME";
const SYMBOL = "SYMBOL";
const TOKEN_ID = 1234;
const EXPIRATION = 2000000000;
const YEAR = 31536000;

describe("ERC7507", function () {

  async function deployContractFixture() {
    const [deployer, owner, user1, user2] = await ethers.getSigners();

    const contract = await ethers.deployContract("ERC7507", [NAME, SYMBOL], deployer);
    await contract.mint(owner, TOKEN_ID);

    return { contract, owner, user1, user2 };
  }

  describe("Functions", function () {
    it("Should not set user if not owner or approved", async function () {
      const { contract, user1 } = await loadFixture(deployContractFixture);

      await expect(contract.setUser(TOKEN_ID, user1, EXPIRATION))
        .to.be.revertedWith("ERC7507: caller is not owner or approved");
    });

    it("Should return zero expiration for nonexistent user", async function () {
      const { contract, user1 } = await loadFixture(deployContractFixture);

      expect(await contract.userExpires(TOKEN_ID, user1)).to.equal(0);
    });

    it("Should set users and then update", async function () {
      const { contract, owner, user1, user2 } = await loadFixture(deployContractFixture);

      await contract.connect(owner).setUser(TOKEN_ID, user1, EXPIRATION);
      await contract.connect(owner).setUser(TOKEN_ID, user2, EXPIRATION);

      expect(await contract.userExpires(TOKEN_ID, user1)).to.equal(EXPIRATION);
      expect(await contract.userExpires(TOKEN_ID, user2)).to.equal(EXPIRATION);

      await contract.connect(owner).setUser(TOKEN_ID, user1, EXPIRATION + YEAR);
      await contract.connect(owner).setUser(TOKEN_ID, user2, 0);

      expect(await contract.userExpires(TOKEN_ID, user1)).to.equal(EXPIRATION + YEAR);
      expect(await contract.userExpires(TOKEN_ID, user2)).to.equal(0);
    });
  });

  describe("Events", function () {
    it("Should emit event when set user", async function () {
      const { contract, owner, user1 } = await loadFixture(deployContractFixture);

      await expect(contract.connect(owner).setUser(TOKEN_ID, user1, EXPIRATION))
        .to.emit(contract, "UpdateUser").withArgs(TOKEN_ID, user1.address, EXPIRATION);
    });
  });

});
```

## Reference Implementation

Reference implementation available at: [`ERC7507.sol`](./assets/contracts/ERC7507.sol):

```solidity
// SPDX-License-Identifier: CC0-1.0

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

import "./IERC7507.sol";

contract ERC7507 is ERC721, IERC7507 {

    mapping(uint256 => mapping(address => uint64)) private _expires;

    constructor(
        string memory name, string memory symbol
    ) ERC721(name, symbol) {}

    function supportsInterface(
        bytes4 interfaceId
    ) public view virtual override returns (bool) {
        return interfaceId == type(IERC7507).interfaceId || super.supportsInterface(interfaceId);
    }

    function userExpires(
        uint256 tokenId, address user
    ) public view virtual override returns(uint256) {
        require(_exists(tokenId), "ERC7507: query for nonexistent token");
        return _expires[tokenId][user];
    }

    function setUser(
        uint256 tokenId, address user, uint64 expires
    ) public virtual override {
        require(_isApprovedOrOwner(_msgSender(), tokenId), "ERC7507: caller is not owner or approved");
        _expires[tokenId][user] = expires;
        emit UpdateUser(tokenId, user, expires);
    }

}
```

## Security Considerations

No security considerations found.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
