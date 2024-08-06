1. If the forge() mints the last tokenId in the gen, next user could receive the tokenId in the gen+1 with the significant less price, which lead to the protocol money loose.
## Impact
- The problem here is that if the mintEntity will be minted exactly at the end of the generation, the next user could mint an NFT without paying a full price for it, only the base price.
Once the gen will be updated, the generationMintCounts will be set to 0.
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L354
After, the user will invoke the mintToken function and the price will be multiplied by 0, which will lead to more favorable price the user can buy a token https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L181
##Recommended Mitigation Steps
Ensure that the price for the first token minted will not be multiplied by 0


## Impact
2. no check in the _incrementGeneration that ensures there is only 10 gen.

## Impact
3. mintWithBudget doesn’t ensure the msg.sender as payable

## Impact
4. During the forgeWithListed function , the excess fees is not refunded