---
eip: 5269
title: ERC Detection and Discovery
description: An interface to identify if major behavior or optional behavior specified in an ERC is supported for a given caller.
author: Zainan Victor Zhou (@xinbenlv)
discussions-to: https://ethereum-magicians.org/t/erc5269-human-readable-interface-detection/9957
status: Review
type: Standards Track
category: ERC
created: 2022-07-15
requires: 5750
---

## Abstract

An interface for better identification and detection of ERC by numbers.
It designates a field in which it's called `majorERCIdentifier` which is normally known or referred to as "ERC number". For example, `ERC-721` aka [ERC-721](../00721.md) has a `majorERCIdentifier = 721`. This ERC has a `majorERCIdentifier = 5269`.

Calling it a `majorERCIdentifier` instead of `ERCNumber` makes it future-proof: anticipating there is a possibility where future ERC is not numbered or if we want to incorporate other types of standards.

It also proposes a new concept of `minorERCIdentifier` which is left for authors of
individual ERC to define. For example, ERC-721's author may define `ERC721Metadata`
interface as `minorERCIdentifier= keccak256("ERC721Metadata")`.

It also proposes an event to allow smart contracts to optionally declare the ERCs they support.

## Motivation

This ERC is created as a competing standard for [ERC-165](../00165.md).

Here are the major differences between this ERC and [ERC-165](../00165.md).

1. [ERC-165](../00165.md) uses the hash of a method's signature which declares the existence of one method or multiple methods,
therefore it requires at least one method to *exist* in the first place. In some cases, some ERCs interface does not have a method, such as some ERCs related to data format and signature schemes or the "Soul-Bound-ness" aka SBT which could just revert a transfer call without needing any specific method.
1. [ERC-165](../00165.md) doesn't provide query ability based on the caller.
The compliant contract of this ERC will respond to whether it supports certain ERC *based on* a given caller.

Here is the motivation for this ERC given ERC-165 already exists:

1. Using ERC numbers improves human readability as well as make it easier to work with named contract such as ENS.

2. Instead of using an ERC-165 identifier, we have seen an increasing interest to use ERC numbers as the way to identify or specify an ERC. For example

- [ERC-5267](../05267.md) specifies `extensions` to be a list of ERC numbers.
- [ERC-600](../00600.md), and [ERC-601](../00601.md) specify an `ERC` number in the `m / purpose' / subpurpose' / ERC' / wallet'` path.
- [ERC-5568](../05568.md) specifies `The instruction_id of an instruction defined by an ERC MUST be its ERC number unless there are exceptional circumstances (be reasonable)`
- [ERC-6120](../06120.md) specifies `struct Token { uint eip; ..., }` where `uint eip` is an ERC number to identify ERCs.
- `ERC-867`(Stagnant) proposes to create `erpId: A string identifier for this ERP (likely the associated ERC number, e.g. “ERC-1234”).`

3. Having an ERC/ERC number detection interface reduces the need for a lookup table in smart contract to
convert a function method or whole interface in any ERC in the bytes4 ERC-165 identifier into its respective ERC number and massively simplifies the way to specify ERC for behavior expansion.

4. We also recognize a smart contract might have different behavior given different caller accounts. One of the most notable use cases is that when using Transparent Upgradable Pattern, a proxy contract gives an Admin account and Non-Admin account different treatment when they call.

## Specification

In the following description, we use ERC and ERC inter-exchangeably. This was because while most of the time the description applies to an ERC category of the Standards Track of ERC, the ERC number space is a subspace of ERC number space and we might sometimes encounter ERCs that aren't recognized as ERCs but has behavior that's worthy of a query.

1. Any compliant smart contract MUST implement the following interface

```solidity
// DRAFTv1
pragma solidity ^0.8.9;

interface IERC5269 {
  event OnSupportERC(
      address indexed caller, // when emitted with `address(0x0)` means all callers.
      uint256 indexed majorERCIdentifier,
      bytes32 indexed minorERCIdentifier, // 0 means the entire ERC
      bytes32 ercStatus,
      bytes extraData
  );

  /// @dev The core method of ERC Interface Detection
  /// @param caller, a `address` value of the address of a caller being queried whether the given ERC is supported.
  /// @param majorERCIdentifier, a `uint256` value and SHOULD BE the ERC number being queried. Unless superseded by future ERC, such ERC number SHOULD BE less or equal to (0, 2^32-1]. For a function call to `supportERC`, any value outside of this range is deemed unspecified and open to implementation's choice or for future ERCs to specify.
  /// @param minorERCIdentifier, a `bytes32` value reserved for authors of individual ERC to specify. For example the author of [ERC-721](/ERCS/eip-721) MAY specify `keccak256("ERC721Metadata")` or `keccak256("ERC721Metadata.tokenURI")` as `minorERCIdentifier` to be quired for support. Author could also use this minorERCIdentifier to specify different versions, such as ERC-712 has its V1-V4 with different behavior.
  /// @param extraData, a `bytes` for [ERC-5750](/ERCS/eip-5750) for future extensions.
  /// @return ercStatus, a `bytes32` indicating the status of ERC the contract supports.
  ///                    - For FINAL ERCs, it MUST return `keccak256("FINAL")`.
  ///                    - For non-FINAL ERCs, it SHOULD return `keccak256("DRAFT")`.
  ///                      During ERC procedure, ERC authors are allowed to specify their own
  ///                      ercStatus other than `FINAL` or `DRAFT` at their discretion such as `keccak256("DRAFTv1")`
  ///                      or `keccak256("DRAFT-option1")`and such value of ercStatus MUST be documented in the ERC body
  function supportERC(
    address caller,
    uint256 majorERCIdentifier,
    bytes32 minorERCIdentifier,
    bytes calldata extraData)
  external view returns (bytes32 ercStatus);
}
```

In the following description, `ERC_5269_STATUS` is set to be `keccak256("DRAFTv1")`.

In addition to the behavior specified in the comments of `IERC5269`:

1. Any `minorERCIdentifier=0` is reserved to be referring to the main behavior of the ERC being queried.
2. The Author of compliant ERC is RECOMMENDED to declare a list of `minorERCIdentifier` for their optional interfaces, behaviors and value range for future extension.
3. When this ERC is FINAL, any compliant contract MUST return an `ERC_5269_STATUS` for the call of `supportERC((any caller), 5269, 0, [])`

*Note*: at the current snapshot, the `supportERC((any caller), 5269, 0, [])` MUST return `ERC_5269_STATUS`.

4. Any complying contract SHOULD emit an `OnSupportERC(address(0), 5269, 0, ERC_5269_STATUS, [])` event upon construction or upgrade.
5. Any complying contract MAY declare for easy discovery any ERC main behavior or sub-behaviors by emitting an event of `OnSupportERC` with relevant values and when the compliant contract changes whether the support an ERC or certain behavior for a certain caller or all callers.
6. For any `ERC-XXX` that is NOT in `Final` status, when querying the `supportERC((any caller), xxx, (any minor identifier), [])`, it MUST NOT return `keccak256("FINAL")`. It is RECOMMENDED to return `0` in this case but other values of `ercStatus` is allowed. Caller MUST treat any returned value other than `keccak256("FINAL")` as non-final, and MUST treat 0 as strictly "not supported".
7. The function `supportERC` MUST be mutability `view`, i.e. it MUST NOT mutate any global state of EVM.

## Rationale

1. When data type `uint256 majorERCIdentifier`, there are other alternative options such as:

- (1) using a hashed version of the ERC number,
- (2) use a raw number, or
- (3) use an ERC-165 identifier.

The pros for (1) are that it automatically supports any evolvement of future ERC numbering/naming conventions.
But the cons are it's not backward readable: seeing a `hash(ERC-number)` one usually can't easily guess what their ERC number is.

We choose the (2) in the rationale laid out in motivation.

2. We have a `bytes32 minorERCIdentifier` in our design decision. Alternatively, it could be (1) a number, forcing all ERC authors to define its numbering for sub-behaviors so we go with a `bytes32` and ask the ERC authors to use a hash for a string name for their sub-behaviors which they are already doing by coming up with interface name or method name in their specification.

3. Alternatively, it's possible we add extra data as a return value or an array of all ERC being supported but we are unsure how much value this complexity brings and whether the extra overhead is justified.

4. Compared to [ERC-165](../00165.md), we also add an additional input of `address caller`, given the increasing popularity of proxy patterns such as those enabled by [ERC-1967](../01967.md). One may ask: why not simply use `msg.sender`? This is because we want to allow query them without transaction or a proxy contract to query whether interface ERC-`number` will be available to that particular sender.

1. We reserve the input `majorERCIdentifier` greater than or equals `2^32` in case we need to support other collections of standards which is not an ERC/ERC.

## Test Cases

```typescript

describe("ERC5269", function () {
  async function deployFixture() {
    // ...
  }

  describe("Deployment", function () {
    // ...
    it("Should emit proper OnSupportERC events", async function () {
      let { txDeployErc721 } = await loadFixture(deployFixture);
      let events = txDeployErc721.events?.filter(event => event.event === 'OnSupportERC');
      expect(events).to.have.lengthOf(4);

      let ev5269 = events!.filter(
        (event) => event.args!.majorERCIdentifier.eq(5269));
      expect(ev5269).to.have.lengthOf(1);
      expect(ev5269[0].args!.caller).to.equal(BigNumber.from(0));
      expect(ev5269[0].args!.minorERCIdentifier).to.equal(BigNumber.from(0));
      expect(ev5269[0].args!.ercStatus).to.equal(ethers.utils.id("DRAFTv1"));

      let ev721 = events!.filter(
        (event) => event.args!.majorERCIdentifier.eq(721));
      expect(ev721).to.have.lengthOf(3);
      expect(ev721[0].args!.caller).to.equal(BigNumber.from(0));
      expect(ev721[0].args!.minorERCIdentifier).to.equal(BigNumber.from(0));
      expect(ev721[0].args!.ercStatus).to.equal(ethers.utils.id("FINAL"));

      expect(ev721[1].args!.caller).to.equal(BigNumber.from(0));
      expect(ev721[1].args!.minorERCIdentifier).to.equal(ethers.utils.id("ERC721Metadata"));
      expect(ev721[1].args!.ercStatus).to.equal(ethers.utils.id("FINAL"));

      // ...
    });

    it("Should return proper ercStatus value when called supportERC() for declared supported ERC/features", async function () {
      let { erc721ForTesting, owner } = await loadFixture(deployFixture);
      expect(await erc721ForTesting.supportERC(owner.address, 5269, ethers.utils.hexZeroPad("0x00", 32), [])).to.equal(ethers.utils.id("DRAFTv1"));
      expect(await erc721ForTesting.supportERC(owner.address, 721, ethers.utils.hexZeroPad("0x00", 32), [])).to.equal(ethers.utils.id("FINAL"));
      expect(await erc721ForTesting.supportERC(owner.address, 721, ethers.utils.id("ERC721Metadata"), [])).to.equal(ethers.utils.id("FINAL"));
      // ...

      expect(await erc721ForTesting.supportERC(owner.address, 721, ethers.utils.id("WRONG FEATURE"), [])).to.equal(BigNumber.from(0));
      expect(await erc721ForTesting.supportERC(owner.address, 9999, ethers.utils.hexZeroPad("0x00", 32), [])).to.equal(BigNumber.from(0));
    });

    it("Should return zero as ercStatus value when called supportERC() for non declared ERC/features", async function () {
      let { erc721ForTesting, owner } = await loadFixture(deployFixture);
      expect(await erc721ForTesting.supportERC(owner.address, 721, ethers.utils.id("WRONG FEATURE"), [])).to.equal(BigNumber.from(0));
      expect(await erc721ForTesting.supportERC(owner.address, 9999, ethers.utils.hexZeroPad("0x00", 32), [])).to.equal(BigNumber.from(0));
    });
  });
});
```

See [`TestERC5269.ts`](./assets/test/TestERC5269.ts).

## Reference Implementation

Here is a reference implementation for this ERC:

```solidity
contract ERC5269 is IERC5269 {
    bytes32 constant public ERC_STATUS = keccak256("DRAFTv1");
    constructor () {
        emit OnSupportERC(address(0x0), 5269, bytes32(0), ERC_STATUS, "");
    }

    function _supportERC(
        address /*caller*/,
        uint256 majorERCIdentifier,
        bytes32 minorERCIdentifier,
        bytes calldata /*extraData*/)
    internal virtual view returns (bytes32 ercStatus) {
        if (majorERCIdentifier == 5269) {
            if (minorERCIdentifier == bytes32(0)) {
                return ERC_STATUS;
            }
        }
        return bytes32(0);
    }

    function supportERC(
        address caller,
        uint256 majorERCIdentifier,
        bytes32 minorERCIdentifier,
        bytes calldata extraData)
    external virtual view returns (bytes32 ercStatus) {
        return _supportERC(caller, majorERCIdentifier, minorERCIdentifier, extraData);
    }
}
```

See [`ERC5269.sol`](./assets/contracts/ERC5269.sol).

Here is an example where a contract of [ERC-721](../00721.md) also implement this ERC to make it easier
to detect and discover:

```solidity
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "../ERC5269.sol";
contract ERC721ForTesting is ERC721, ERC5269 {

    bytes32 constant public ERC_FINAL = keccak256("FINAL");
    constructor() ERC721("ERC721ForTesting", "E721FT") ERC5269() {
        _mint(msg.sender, 0);
        emit OnSupportERC(address(0x0), 721, bytes32(0), ERC_FINAL, "");
        emit OnSupportERC(address(0x0), 721, keccak256("ERC721Metadata"), ERC_FINAL, "");
        emit OnSupportERC(address(0x0), 721, keccak256("ERC721Enumerable"), ERC_FINAL, "");
    }

  function supportERC(
    address caller,
    uint256 majorERCIdentifier,
    bytes32 minorERCIdentifier,
    bytes calldata extraData)
  external
  override
  view
  returns (bytes32 ercStatus) {
    if (majorERCIdentifier == 721) {
      if (minorERCIdentifier == 0) {
        return keccak256("FINAL");
      } else if (minorERCIdentifier == keccak256("ERC721Metadata")) {
        return keccak256("FINAL");
      } else if (minorERCIdentifier == keccak256("ERC721Enumerable")) {
        return keccak256("FINAL");
      }
    }
    return super._supportERC(caller, majorERCIdentifier, minorERCIdentifier, extraData);
  }
}

```

See [`ERC721ForTesting.sol`](./assets/contracts/testing/ERC721ForTesting.sol).

## Security Considerations

Similar to [ERC-165](../00165.md) callers of the interface MUST assume the smart contract
declaring they support such ERC interfaces doesn't necessarily correctly support them.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
