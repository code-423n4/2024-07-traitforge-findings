| Severity | Title | 
|:--:|:--:|
| [L-01](#L-01) | TraitForgeNft::mintWithBudWithBudget() will revert when users try to mint NFTs with ids bigger than **10_000**|
| [L-02](#L-02) | Functions and variables that won't be used should be removed|
| [L-03](#L-03) | Users that call EntityForging::forgeWithListed() in order to forge a new NFT, may send more ETH than required, that ETH will be stuck in the contract forever.|
| [L-04](#L-04) | No setter for TraitForgeNft::maxTokensPerGen parameter.|
| [L-05](#L-05) | The TraitForgeNft::getTokenGeneration function doesn't check if a NFT token exists|
| [L-06](#L-06) | EntityTrading::buyNFT() function charges 10% of the fee, and sends it to the NukeFund, however there are no restrictions that NFTs can be sold only via that contract. |
| [L-07](#L-07) | A single whitelisted user can mint all NFTs |


# [L-01] TraitForgeNft::mintWithBudWithBudget() will revert when users try to mint NFTs with ids bigger than **10_000**
The [mintWithBudget()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225) function is used to allow users to mint multiple NFTs within a single transaction. However there is a problem within the while loop. As can be seen from the below code snippet, the condition in the while loop requires that the ``_tokenIds < maxTokensPerGen``, **maxTokensPerGen** is set to **10_000** and cannot be changed. However **_tokenIds** increases with each minted NFT. When the first generation of NFTs is minted, the [mintWithBudget()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225) function will always revert. 

```solidity
  function mintWithBudget(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
    uint256 mintPrice = calculateMintPrice();
    uint256 amountMinted = 0;
    uint256 budgetLeft = msg.value;

    while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
      _mintInternal(msg.sender, mintPrice);
      amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
    if (budgetLeft > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: budgetLeft }('');
      require(refundSuccess, 'Refund failed.');
    }
  }
```
### Recommended Mitigation Steps
Consider removing the ``_tokenIds < maxTokensPerGen`` from the while loop.

# [L-02] Functions and variables that won't be used should be removed
Remove the following functions and variables:
 - [ageMultiplier;](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/NukeFund/NukeFund.sol#L24) should be removed as it won't be used. 
 - [setAgeMultplier()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/NukeFund/NukeFund.sol#L109-L111)
 - [getAgeMultiplier()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/NukeFund/NukeFund.sol#L113-L115)
 - Also make sure to remove **ageMultiplier** from the formula for calculating age in the [calculateAge()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/NukeFund/NukeFund.sol#L118-L133) function
 - [deriveTokenParameters()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntropyGenerator/EntropyGenerator.sol#L136-L161)
 - [getFirstDigit()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntropyGenerator/EntropyGenerator.sol#L198-L203)

The above listed functions and parameters have been confirmed by the sponsor in the public chat of the context, that won't be used, this can also be seen from the deployments scripts and test files. 

# [L-03] Users that call EntityForging::forgeWithListed() in order to forge a new NFT, may send more ETH than required, that ETH will be stuck in the contract forever.
In the ``EntityForging.sol`` contract, users can forge a new NFT by calling the [forgeWithListed()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntityForging/EntityForging.sol#L102-L175) function, they have to pay a fee set by the owner of the NFT they are forging with. As we can see from the following check, the amount that a user pays may be bigger than the required fee:
```solidity
  function forgeWithListed(uint256 forgerTokenId, uint256 mergerTokenId) external payable whenNotPaused nonReentrant returns (uint256) {
    Listing memory _forgerListingInfo = listings[listedTokenIds[forgerTokenId]];
    require(_forgerListingInfo.isListed, "Forger's entity not listed for forging");
    require(nftContract.ownerOf(mergerTokenId) == msg.sender, 'Caller must own the merger token');
    require(nftContract.ownerOf(forgerTokenId) != msg.sender, 'Caller should be different from forger token owner');
    require(nftContract.getTokenGeneration(mergerTokenId) == nftContract.getTokenGeneration(forgerTokenId), 'Invalid token generation');

    uint256 forgingFee = _forgerListingInfo.fee;
    require(msg.value >= forgingFee, 'Insufficient fee for forging');

    _resetForgingCountIfNeeded(forgerTokenId); // Reset for forger if needed
    _resetForgingCountIfNeeded(mergerTokenId); // Reset for merger if needed

    // Check forger's breed count increment but do not check forge potential here
    // as it is already checked in listForForging for the forger
    forgingCounts[forgerTokenId]++;

    // Check and update for merger token's forge potential
    uint256 mergerEntropy = nftContract.getTokenEntropy(mergerTokenId);
    require(mergerEntropy % 3 != 0, 'Not merger');
    uint8 mergerForgePotential = uint8((mergerEntropy / 10) % 10); // Extract the 5th digit from the entropy
    forgingCounts[mergerTokenId]++;
    require(mergerForgePotential > 0 && forgingCounts[mergerTokenId] <= mergerForgePotential, 'forgePotential insufficient');

    uint256 devFee = forgingFee / taxCut;
    uint256 forgerShare = forgingFee - devFee;
    address payable forgerOwner = payable(nftContract.ownerOf(forgerTokenId));

    uint256 newTokenId = nftContract.forge(msg.sender, forgerTokenId, mergerTokenId, '');
    (bool success, ) = nukeFundAddress.call{ value: devFee }('');
    require(success, 'Failed to send to NukeFund');
    (bool success_forge, ) = forgerOwner.call{ value: forgerShare }('');
    require(success_forge, 'Failed to send to Forge Owner');

    ...
  }
```
As we can see, the **forgingFee** is being sent to the NukeFund and to the forgerOwner, not the msg.value and there is no functionality that checks if an excess amount of ETH was sent, and returns the excess amount to the msg.sender.This may result in ETH being stuck in the ``EntityForging.sol`` contract permanently as there is no function that withdraws an arbitrary amount of EHT to an arbitrary address.
This is contrary to functions such as [mintToken()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L181-L200) and [mintWithBudget()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225) which check if an excess amount of ETH was sent, and return it to the msg.sender. 

### Recommended Mitigation Steps
In the [forgeWithListed()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntityForging/EntityForging.sol#L102-L175) function check if an excess amount was sent, and if so return it to the msg.sender. This mechanism already exists in the [mintToken()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L181-L200) and [mintWithBudget()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225) funcions.

# [L-04] No setter for TraitForgeNft::maxTokensPerGen parameter.
In the ``TraitForgeNft.sol`` contract there is no setter function for the maxTokensPerGen parameter. If the owners of the contract decide to increase or decrease the NFTs that can be minted in each new generation, they can't do it.

### Recommended Mitigation Steps
Implement a setter function which can set the maxTokensPerGen parameter. This function should only be callable by the owner of the contract. Consider whether the setter function should have certain restrictions to the number to which it can be set, and when it can be set, and whether the new maxTokensPerGen parameter should affect already minted generations. For example only set it when the contracts are paused. 

# [L-05] The TraitForgeNft::getTokenGeneration function doesn't check if a NFT token exists.
In the ``TraitForgeNft.sol`` contract the [getTokenGeneration()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L242-L244) function returns the generation of a certain token, however the function doesn't check whether such a token exists, it may return a result for a token that has already been burned or for a token that is not yet minted. 

### Recommended Mitigation Steps
Before returning a value check whether a token with such id exists

# [L-06] EntityTrading::buyNFT() function charges 10% of the fee, and sends it to the NukeFund, however there are no restrictions that NFTs can be sold only via that contract. 
The [EntityTrading::buyNFT()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntityTrading/EntityTrading.sol#L63-L92) function charges 10% on each sale, and sends it to the NukeFund contract. However there are no restrictions that requires the users to list their NFTs for sale only via the EntityTrading contract. They may decide to use other NFT marketplaces platforms like OpenSea and try to avoid the selling fee. This may affect the economics of the game as the NukeFund is used for paying rewards to users that burn their NFTs, if users sell their NFTs via a third party marketplace 10% less fees will be received by the NukeFund contract. There is also the possibility that if a user has listed his NFT for forging via the [listForForging()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntityForging/EntityForging.sol#L67-L100) function, his listing won't be canceled until the NFT is bought, and someone can  potentially forge with it before the NFT is bough, and before the protocol tries to call the [cancelListingForForging()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/EntityForging/EntityForging.sol#L177-L191) via the [_beforeTokenTransfer()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L367-L392) function. Keep in mind that each NFT has a **forgePotential** that determines how much times each NFT can be listed for forging and thus collect a certain fee for it. There is a mapping **mapping(uint256 => uint8) public forgingCounts;** that tracks how much time each NFT was used for forging, the number of times an NFT can be used for forging impacts the price of the NFT, as it directly relates to the profit each NFT can generate via the in game mechanics. Thus if someone forges with an NFT, before it is bought on another marketplace, the buyer of the NFT will effectively be robbed of certain fees, that he could have received otherwise(if he was the one that listed the NFT for forging). Keep in mind this doesn't require forntruniing, as simply the transaction of another user to forge with a certain NFT can be received by the sequencer right before the transaction of the buyer to buy the NFT.
 
### Recommended Mitigation Steps
It is hard to come up with a fix, as there is no simple way to require users to use a specific marketplace. Consider whether you need to charge 10% on each sale done via the EntityTrading contract, and make sure to inform the potential buyers that if the owner of the NFT has listed his NFT for forging, someone can forge with it before it is bough from another market platform. 

# [L-07] A single whitelisted user can mint all NFTs
The ``TraitForgeNft.sol`` contracts implement a whitelisting mechanism to allow only whitelisted users to mint NFTs in some initial duration after the protocol has been deployed. Usually whitelisted mechanisms are implemented in order to reward some users with the possibility of early mints, and to prevent a couple of users to accumulate most of the supply of the asset being offered for sale. However each whitelisted user can mint as much NFTs as he desires. This may lead to different problems, for example if 10 users mint all of the NFTs in the 10 generations, and there are 1000 whitelisted users, 990 users won't be able to mint NFTs during the whitelisting period. This results in a very poor experience for the users that couldn't mint. Also if whales accumulate most of the supply of NFTs they may try to manipulate the price of NFTs on the open market. 

### Recommended Mitigation Steps
Consider implementing a whitelisting mechanism, which allows users to mint only a specific predefined number of NFTs. In [mintWithBudget()](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225) implement additional safety checks to ensure the specified amount of NFTs is not surpassed. 