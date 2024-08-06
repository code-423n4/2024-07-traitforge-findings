
## L-01 The `TraitForgeNft.mintToken()` function should utilize `payable(msg.sender)` instead of `msg.sender` when refunding excess Ether paid.

When minting a new NFT, any excess Ether paid is refunded to the caller at `L197` by executing `msg.sender.call{value: excessPayment}('')`. However, if `msg.sender` is a non-payable contract, this will cause a revert. It is recommended to use `payable(msg.sender)` instead of `msg.sender` for better safety.

https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L181-L200

```solidity
    function mintToken(
      ...

197     (bool refundSuccess, ) = msg.sender.call{ value: excessPayment }('');
      ...
```

## L-02 The `TraitForgeNft.getTokenGeneration()` function should revert when called for non-existent NFTs.

If the `tokenId` corresponds to a burnt NFT, the value of `tokenGenerations[tokenId]` remains unchanged. Therefore, the `getTokenGeneration()` function returns this unchanged value, as if the `tokenId` were still an active NFT.

https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L242-L244

```solidity
    function getTokenGeneration(uint256 tokenId) public view returns (uint256) {
243   return tokenGenerations[tokenId];
    }
```

## L-03 The `NukeFund.nuke()` function should reduce `generationMintCounts` for the generation of the `tokenId`.

In the `nuke()` function, it burns the `tokenId`. As a result, the actual number of NFTs that have the same generation as the burnt `tokenId` is decreased by 1.

https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/NukeFund/NukeFund.sol#L153-L182

```solidity
    function nuke(uint256 tokenId) public whenNotPaused nonReentrant {
        ...

176     nftContract.burn(tokenId); // Burn the token
        ...
    }
```

## L-04 The `TraitForgeNft.setMaxGeneration()` function should check if there are any NFTs with a generation greater than the newly set `maxGeneration`.

In the `setMaxGeneration()` function, it only checks if `maxGeneration_` is not less than `currentGeneration`. However, if `maxGeneration_` equals `currentGeneration` and there are NFTs in the `currentGeneration + 1` generation (could be minted by forging), setting `maxGeneration` to `currentGeneration` allows some NFTs with a generation greater than `maxGeneration`.

https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L114-L120

```solidity
    function setMaxGeneration(uint maxGeneration_) external onlyOwner {
      require(
116     maxGeneration_ >= currentGeneration,
        "can't below than current generation"
      );
      maxGeneration = maxGeneration_;
    }
```

## L-05 `entropySlots` are predetermined, which could potentially cause a mint race.

Users can mint any NFT they desire or choose to avoid minting those they don't want. Additionally, a malicious user can interrupt others if they are about to mint a desirable NFT, or delay others' minting processes, potentially causing the minting of NFTs that others do not want, through front-running.

https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntropyGenerator/EntropyGenerator.sol#L47-L98

## L-06 The `deriveTokenParameters()` function returns an incorrect `nukeFactor`.

It consistently calculates the `nukeFactor` as 0 because entropy is less than 1,000,000. It is recommended to divide by 40 instead of 4,000,000.

https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntropyGenerator/EntropyGenerator.sol#L136-L161

```solidity
    function deriveTokenParameters(
      ...

152   nukeFactor = entropy / 4000000;
      ...
```

## L-07 The `EntityForging.lastForgeResetTimestamp` variable should be public.

This variable is used to reset the forging count of NFT token.
If forging count is full, the owner of token needs to get `lastForgeResetTimestamp` value to know when the token is available to forge
But `lastForgeResetTimestamp` is private variable from L23

https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntityForging/EntityForging.sol#L23

```solidity
L23:        mapping(uint256 => uint256) private lastForgeResetTimestamp;
```

`lastForgeResetTimestamp` should be public variable.

## L-08 Potential DoS in the `EntityForging::fetchListings` function

`listingCount` is always increased but never decreased and it could be the source of the DoS in `EntityForging::fetchListings` function if other apps or contracts utilize this function because of the long loop.

```solidity

EntityForging.sol
48: function fetchListings() external view returns (Listing[] memory _listings) {
49:     _listings = new Listing[](listingCount + 1);
50:     for (uint256 i = 1; i <= listingCount; ++i) {
51:       _listings[i] = listings[i];
52:     }
53:   }

```