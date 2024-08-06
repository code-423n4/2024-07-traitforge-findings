# Summary
| #   | Issue|
| ----| ------- |
| 01| `nukeFactor` in `EntropyGenerator::deriveTokenParameters` incorrectly returns 0 due to decimal truncation, affecting player's strategy in gameplay  |
| 02| `forgePotential` in `EntropyGenerator::deriveTokenParameters` returns incorrect result, affecting player's strategy in gameplay|
| 03| Incorrect timestamp reset for `forgePotential`     |
| 04| Weak source of randomness in deriving entropy     |
| 05| URIStorage is not inherited     |
| 06| `maxGeneration` check included for forging a new NFT but not included when minting a new NFT |

# Details

## [01] `nukeFactor` in `EntropyGenerator::deriveTokenParameters` incorrectly returns 0 due to decimal truncation, affecting player's strategy in gameplay

According to the [gitbook](https://github.com/TraitForge/GitBook/blob/main/GamePlay/Nuke%20Fund.md), entity holders can "nuke" their entities, which involves calculating a "nuke factor" based on the entities' age and entropy. This factor determines how much of the nuke fund the entity holder can claim. 

In `EntropyGenerator.sol`, the variable `nukeFactor` is calculated as [follows](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L152):

```js
 nukeFactor = entropy / 4000000;
``` 

Note that the value of entropy in this case returns 6 digits. See  `EntropyGenerator.sol::function getEntropy` extracted [below](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L164-L185):
 
```js
function getEntropy(
    uint256 slotIndex,
    uint256 numberIndex
  ) private view returns (uint256) {
    require(slotIndex <= maxSlotIndex, 'Slot index out of bounds.');

    if (
      slotIndex == slotIndexSelectionPoint &&
      numberIndex == numberIndexSelectionPoint
    ) {
      return 999999;
    }

    uint256 position = numberIndex * 6; // calculate the position for slicing the entropy value
    require(position <= 72, 'Position calculation error');

    uint256 slotValue = entropySlots[slotIndex]; // slice the required [art of the entropy value
    uint256 entropy = (slotValue / (10 ** (72 - position))) % 1000000; // adjust the entropy value based on the number of digits
    uint256 paddedEntropy = entropy * (10 ** (6 - numberOfDigits(entropy)));

    return paddedEntropy; // return the caculated entropy value
  }
```

Given that the `entropy` value is only 6 digits, the result of `nukeFactor` from this division will be a value less than 1 and therefore truncated to 0. This returns the incorrect `nukeFactor` of the NFT to the caller, affecting player's strategy in this game.

### Recommended Mitigation Steps
Update the formula to determine `nukeFactor` using the correct scale. 

## [02] `forgePotential` in `EntropyGenerator::deriveTokenParameters` returns incorrect result, affecting player's strategy in gameplay

According to the [docs](https://docs.google.com/document/d/1pihtkKyyxobFWdaNU4YfAy56Q7WIMbFJjSHUAfRm6BA/edit), Forge Potential dictates how many times an Entity can forge before becoming infertile for the rest of the year. In the docs and code in [EntityForging.sol](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L86), the forge potential is presented to be derived from the 5th digit of the entropy. 

The discrepancy noted here is that `EntropyGenerator::deriveTokenParameters` caluclates the forge potential to be the [first digit of the entopy](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L153), which is incorrect and affects player's strategy in this game:

```js
function deriveTokenParameters(
    uint256 slotIndex,
    uint256 numberIndex
  )
    public
    view
    returns (
      uint256 nukeFactor,
      uint256 forgePotential,
      uint256 performanceFactor,
      bool isForger
    )
  ...
    forgePotential = getFirstDigit(entropy);
```

### Recommended Mitigation Steps
Update the formula in `EntropyGenerator::deriveTokenParameters` to derive `forgePotential` from the 5th digit of the entropy instead. 

## [03] Incorrect timestamp reset for forgePotential 

Based on the [gitbook](https://github.com/TraitForge/GitBook/blob/main/TraitForger%20Entity/Forge%20Potential%20.md), the game has a timestamp from mint that will reset the mapping count (for `forgePotential`) once 1 year has been passed in blocks.

However, based on the code, this timer reset is done via function `_resetForgingCountIfNeeded` only upon [forge listing](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L83C1-L83C31) or [forging](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L128-L129), and not upon creation of new NFT. This undermines the fairness of the game for players. 

### Recommended Mitigation Steps
The timer reset should be included upon creation of new NFT. 

## [04] Weak randomness in deriving entropy 
The `EntropyGenerator::writeEntropyBatch` functions deriving the outcome of the `pseudoRandomValue` relies on weak sources of pseudo-randomness from on-chain attributes (`block.number`), and could be subject to manipulation by a malicious miner.

### Recommended Mitigation Steps
Consider using an external randomness source like Chainlink VRF

## [05] URIStorage is not inherited  
Noted URIStorage was not inherited in the TraitForge contracts. This means the contract would not have the built-in functionality to manage individual token URIs. This causes inflexibility in cases where there is a need to assign unique metadata URIs to individual tokens. It could also result in compatibility concerns if the contract is expected to interact with platforms or dApps that rely on unique token URIs (like marketplaces displaying NFT metadata), thus limiting its usability. 

### Recommended Mitigation Steps
Consider inheriting URIStorage. 

## [06] `maxGeneration` check included for forging a new NFT but not included when minting a new NFT
[TraitForgeNft::forge](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L166) includes the following check to ensure that maxGeneration is not exceeded: 

```js
 require(newGeneration <= maxGeneration, "can't be over max generation");
```
However, this check is not explicitly included in the [TraitForgeNft::mintToken](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L181) , [TraitForgeNft::mintWithBudget](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L202) and [TraitForgeNft::_mintInternal](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280) functions. Without the explicit checks, the `maxGeneration` could be exceed for mints. 

### Recommended Mitigation Steps
Consider including the `maxGeneration` check explicitly for mint functions. 