# [L-01] Listing an NFT for Sale and Canceling Afterwards Prevents Nuking
In the current implementation of the NFT trading and nuking mechanism, listing an NFT for sale followed by cancellation resets the `TokenLastTransferredTimestamp`. This action inadvertently prevents the owner from nuking the token for a period of 3 days, due to the design of the trading functionality. This unintended consequence affects users who listed their NFTs for sale but decided not to sell or transfer them, thereby delaying their ability to nuke their tokens.
```javascript
  function canTokenBeNuked(uint256 tokenId) public view returns (bool) {
    // Ensure the token exists
    require(
      nftContract.ownerOf(tokenId) != address(0),
      'ERC721: operator query for nonexistent token'
    );
@>  uint256 tokenAgeInSeconds = block.timestamp -
@>    nftContract.getTokenLastTransferredTimestamp(tokenId);
@>  // Assuming tokenAgeInSeconds is the age of the token since it's holding the nft, check if it's over minimum days held
@>  return tokenAgeInSeconds >= minimumDaysHeld;
  }
```
## Impact
This issue directly impacts users who wish to nuke their NFTs but have temporarily listed them for sale. It introduces an unnecessary delay and complicates the user experience, potentially discouraging users from participating in the trading aspect of the ecosystem. The core issue lies in the interaction between the trading and nuking functionalities, highlighting a need for a more nuanced approach to handling token listings and cancellations.

## Proof of Concept
The exploit's feasibility is demonstrated through a Foundry test, which outlines the steps taken to list an NFT for sale, cancel the listing, and observe the subsequent inability to nuke the token. Following the cancellation of the listing, the `TokenLastTransferredTimestamp` is reset, preventing the owner from nuking the token for the required period.
```javascript
  function testListingPreventsNuking() public {
    vm.warp(block.timestamp + 24 hours + 1);
    this.mintTokens(1);
    vm.warp(block.timestamp + 3 days);
    assert(nuke.canTokenBeNuked(1));
    nft.setApprovalForAll(address(trade), true);

    trade.listNFTForSale(1, 1 ether);
    trade.cancelListing(1);
    assert(!nuke.canTokenBeNuked(1));
    vm.warp(block.timestamp + 3 days);
    assert(nuke.canTokenBeNuked(1));
  }
```
## Tools Used
Manaul Review, hardhat-foundry
## Recommended Mitigation Steps
To address this issue, consider introducing an exception for transfers when the destination is the trading contract. This adjustment would recognize that listings and cancellations within the trading mechanism do not constitute a meaningful transfer, thus not resetting the `TokenLastTransferredTimestamp`. This change would allow users to list and cancel their NFTs for sale without inadvertently affecting their ability to nuke their tokens.

# [L-02] Ownership Requirement Blocks `TraitForgeNft::_incrementGeneration`

The `TraitForgeNft::_incrementGeneration` function attempts to call the `EntropyGenerator::initializeAlphaIndices` method to set the Golden God NFT of each generation. However, this operation fails because the `TraitForgeNft` contract is not the owner of the `EntropyGenerator` contract. This oversight is compounded by the fact that the ownership transfer is not performed during the deployment script, leaving the `TraitForgeNft` contract unable to execute necessary updates to the `EntropyGenerator`.

```javascript
@> function initializeAlphaIndices() public whenNotPaused onlyOwner {
    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512;
    uint256 numberIndexSelection = hashValue % 13;

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
  }
```
## Impact
The inability to call the `_incrementGeneration` function restricts the `TraitForgeNft` contract's ability to progress generations and assign Golden God NFTs. This limitation effectively halts the advancement of the NFT generation process, impacting the ecosystem's growth and development. Users and stakeholders are affected as new generations of NFTs cannot be introduced, potentially stalling the ecosystem's evolution.

## Tools Used
Manual Review
## Recommended Mitigation Steps
To resolve this issue, consider modifying the `EntropyGenerator::initializeAlphaIndices` function to accept calls from an allowed caller rather than strictly requiring the caller to be the owner. This change would involve replacing the `onlyOwner` modifier with an `onlyAllowedCaller` modifier, specifying the `TraitForgeNft` contract as an allowed caller. This adjustment ensures that the `TraitForgeNft` contract can perform necessary updates to the `EntropyGenerator`, facilitating the progression of generations and the assignment of Golden God NFTs.
```diff
- function initializeAlphaIndices() public whenNotPaused onlyOwner {
+ function initializeAlphaIndices() public whenNotPaused onlyAllowedCaller {

    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512;
    uint256 numberIndexSelection = hashValue % 13;

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
  }
```

# [L-03] Ensuring Safe Transfers in `EntityTrading::buyNFT` and `EntityTrading::cancelListing`
The `EntityTrading` contract employs the `transferFrom` function when transferring an NFT from the owner to the trading contract, which is appropriate for the context. However, when transferring NFTs from the trading contract back to the owner or buyer, the contract should utilize `safeTransferFrom` instead of `transferFrom`. The `safeTransferFrom` function includes an additional check to ensure that the recipient is capable of receiving ERC721 tokens, thereby preventing potential losses of NFTs due to failed transfers.

## Impact
Failure to use `safeTransferFrom` when transferring NFTs from the trading contract to the owner or buyer could lead to lost NFTs. This scenario arises when the recipient is not a valid ERC721 receiver, either due to incorrect setup or other unforeseen reasons. Given the irreversible nature of NFT transfers once confirmed on the blockchain, such losses represent a significant risk to users and the overall trustworthiness of the trading mechanism.

## Tools Used
Manual Review
## Recommended Mitigation Steps
To mitigate this risk, the `EntityTrading` contract should modify its `buyNFT` and `cancelListing` functions to use `safeTransferFrom` for transferring NFTs from the trading contract to the owner or buyer. This change ensures that the contract performs a safety check before attempting the transfer, verifying that the recipient is capable of receiving the NFT.


# [L-04] `EntropyGenerator::intitalizeAlphaIndices` Uses Weak RNG Which Can Be Influenced by Users

The `initializeAlphaIndices` function utilizes a weak random number generator (RNG) in the form of a Keccak256 hash of the block number and block timestamp. Since this information is known to users, a user who is minting the 10,001st NFT, which triggers the `initializeAlphaIndices` function call, can potentially influence the desired selection for index number and slot by choosing a specific time and block.
## Impact
While it is relatively expensive and probably unprofitable for users to exploit this vulnerability, it is advisable to use Chainlink VRF for cases like this to enhance security.

## Proof of Concept
The following foundry test demonstrates how a user could potentially exploit the weakness in the RNG used by the `initializeAlphaIndices` function. By manipulating the block timestamp and number, the user can influence the selection of index numbers and slots, thereby affecting the outcome of the NFT generation process.

```javascript
 function testGetNextEntropy() public {
    vm.deal(user1, 1000 ether);
    vm.warp(block.timestamp + 24 hours + 135); //let whitelist time pass
    vm.roll(block.number + 123);
    // others minted first 10000 tokens of gen 1
    this.mintTokens(10000);
    vm.startPrank(user1);
    while (true) {
      uint256 hashValue = uint256(
        keccak256(
          abi.encodePacked(blockhash(block.number - 1), block.timestamp)
        )
      );
      uint256 slotIndexSelection = (hashValue % 258) + 512;
      // uint256 numberIndexSelection = hashValue % 13;
      if (slotIndexSelection < 513) {
        break;
      }
      vm.warp(block.timestamp + 10);
    }

    //user1 mints 10001 token 1st token of gen 2
    uint256 balanceBefore = user1.balance;
    bytes32[] memory proof;
    //mint 1st token and get the generated slot and number indces
    nft.mintToken{ value: 1 ether }(proof);
    uint256 slotIndexSelectionPoint = entropy.slotIndexSelectionPoint();
    uint256 numberIndexSelectionPoint = entropy.numberIndexSelectionPoint();
    console2.log(slotIndexSelectionPoint);
    console2.log(numberIndexSelectionPoint);
    //using those indices mint untill you reach the desiered god token
    uint256 tokenId = nft.totalSupply();
    for (
      uint256 i;
      i < (slotIndexSelectionPoint + 1) * 13 + numberIndexSelectionPoint;
      i++
    ) {
      if (nft.getTokenEntropy(tokenId) == 999999) {
        break;
      }
      nft.mintToken{ value: 1 ether }(proof);
      tokenId++;
    }
    assertEq(nft.getTokenEntropy(tokenId), 999999);
    console2.log(balanceBefore - user1.balance); //689 102 774 500 000 000 000 /// too much eth needed
    console2.log(tokenId);
  }
  function mintTokens(uint256 iterations) external {
    bytes32[] memory proof;

    for (uint256 i = 0; i < iterations; i++) {
      nft.mintToken{ value: 1000000 ether }(proof);
      // uint256 entropyVal = nft.getTokenEntropy(i + 1);
      // if (entropyVal == 999999) {
      //   console2.log(i + 1);
      // }
    }
  }
```

## Tools Used
Manual Review
## Recommended Mitigation Steps
**Implement Chainlink VRF**: Use Chainlink VRF to generate cryptographically secure random numbers for initializing alpha indices. This ensures that the process is not predictable or manipulatable by users.


# [NC-01] `EntityTrading::buyNFT` Does Not follow CEI
The `EntityTrading::buyNFT` function in the `EntityTrading` contract implements a non-reentrancy guard to prevent reentrant attacks. However, it does not adhere to the Checks-Effects-Interactions (CEI) pattern, a best practice in smart contract development aimed at minimizing potential vulnerabilities and improving code readability and predictability.

## Impact
The absence of the CEI pattern in the `buyNFT` function can lead to subtle bugs and vulnerabilities, especially in complex contracts where multiple functions interact with each other. It can also make the contract harder to understand and maintain, potentially leading to errors during future modifications.

## Tools Used
Manual Review

## Recommended Mitigation Steps
By adhering to the CEI pattern, the `buyNFT` function becomes easier to understand and audit, reducing the risk of subtle bugs and vulnerabilities. This approach also improves the function's predictability and maintainability, making it easier to adapt to future requirements or changes in the contract's logic.
```diff
 function buyNFT(uint256 tokenId) external payable whenNotPaused nonReentrant {
    Listing memory listing = listings[listedTokenIds[tokenId]];
    require(
      msg.value == listing.price,
      'ETH sent does not match the listing price'
    );
    require(listing.seller != address(0), 'NFT is not listed for sale.');

    //transfer eth to seller (distribute to nukefund)
    uint256 nukeFundContribution = msg.value / taxCut;
    uint256 sellerProceeds = msg.value - nukeFundContribution;
    transferToNukeFund(nukeFundContribution); // transfer contribution to nukeFund
+   delete listings[listedTokenIds[tokenId]]; // remove listing
+
+   emit NFTSold(
+     tokenId,
+     listing.seller,
+     msg.sender,
+     msg.value,
+     nukeFundContribution
+   ); // emit an event for the sale
    // transfer NFT from contract to buyer
    (bool success, ) = payable(listing.seller).call{ value: sellerProceeds }(
      ''
    );
    require(success, 'Failed to send to seller');
    nftContract.transferFrom(address(this), msg.sender, tokenId); // transfer NFT to the buyer
-   delete listings[listedTokenIds[tokenId]]; // remove listing

-   emit NFTSold(
-     tokenId,
-     listing.seller,
-     msg.sender,
-     msg.value,
-     nukeFundContribution
-   ); // emit an event for the sale
  }
```

# [NC-02] Unnecessary Requirement in `TraitForgeNFT::_incrementGeneration`
The `_incrementGeneration` function checks if the current generation has reached its maximum count (`maxTokensPerGen`). However, this check is unnecessary because it's already performed before other functions call `_incrementGeneration`.
```javascript
  function _incrementGeneration() private {
@>  require(
@>    generationMintCounts[currentGeneration] >= maxTokensPerGen,
@>    'Generation limit not yet reached'
@>  );
    currentGeneration++;
    generationMintCounts[currentGeneration] = 0; 
    priceIncrement = priceIncrement + priceIncrementByGen; //e this shouldnt be here,
    entropyGenerator.initializeAlphaIndices(); //@audit-written med TraitForgeNFT Cant call the entropy.initializeAlphaIndices because its onlyOwner access control
    emit GenerationIncremented(currentGeneration);
  }
```
```javascript

  function _mintNewEntity(
    address newOwner,
    uint256 entropy,
    uint256 gen
  ) private returns (uint256) {
    require(
      generationMintCounts[gen] < maxTokensPerGen,
      'Exceeds maxTokensPerGen'
    );

    _tokenIds++;
    uint256 newTokenId = _tokenIds;
    _mint(newOwner, newTokenId);

    tokenCreationTimestamps[newTokenId] = block.timestamp;
    tokenEntropy[newTokenId] = entropy;
    tokenGenerations[newTokenId] = gen;
    generationMintCounts[gen]++;
    initialOwners[newTokenId] = newOwner;

@>  if (
@>    generationMintCounts[gen] >= maxTokensPerGen && gen == currentGeneration
@>  ) {
      _incrementGeneration();
    }

    if (!airdropContract.airdropStarted()) {
      airdropContract.addUserAmount(newOwner, entropy);
    }

    emit NewEntityMinted(newOwner, newTokenId, gen, entropy);
    return newTokenId;
  }
```
```javascript
function _mintInternal(address to, uint256 mintPrice) internal {
@>  if (generationMintCounts[currentGeneration] >= maxTokensPerGen) {
@>    _incrementGeneration();
@>  }

```

## Impact
Removing the redundant check will reduce gas costs during transactions involving the incrementation of generations.

## Tools Used
Manual Review

## Recommended Mitigation Steps
- Remove the redundant `require` statement from the `_incrementGeneration` function.
- Ensure that all calls to `_incrementGeneration` are preceded by appropriate checks to avoid unnecessary execution.


# [NC-03] Missing Event Emission When Funds Are Sent to Owner in `NukeFund::receive`
The `receive` function in the `NukeFund` contract checks for three distinct situations and emits events for two of them, omitting an event for one case. This inconsistency leads to incomplete transaction logs and can cause confusion or inaccuracies in tracking the distribution of funds.
```javascript
receive() external payable {
    uint256 devShare = msg.value / taxCut; // Calculate developer's share (10%)
    uint256 remainingFund = msg.value - devShare; // Calculate remaining funds to add to the fund

    fund += remainingFund; // Update the fund balance
    if (!airdropContract.airdropStarted()) {
      (bool success, ) = devAddress.call{ value: devShare }('');
      require(success, 'ETH send failed');
@>    emit DevShareDistributed(devShare);
    } else if (!airdropContract.daoFundAllowed()) {
@>    (bool success, ) = payable(owner()).call{ value: devShare }('');
@>    require(success, 'ETH send failed');
    } else {
      (bool success, ) = daoAddress.call{ value: devShare }('');
      require(success, 'ETH send failed');
@>    emit DevShareDistributed(devShare);
    }

    emit FundReceived(msg.sender, msg.value); // Log the received funds
    emit FundBalanceUpdated(fund); // Update the fund balance
  }
```
## Impact
This inconsistency in event emission can lead to confusion and inaccuracies in tracking the distribution of funds, especially when analyzing transaction histories or auditing the contract.
## tools Used
Manual review
## Recommended Mitigation Steps
- Ensure consistent event emission across all branches of the conditional logic within the `receive` function.
- Add the missing `emit DevShareDistributed(devShare);` line in the branch where it was previously omitted to maintain consistency and accuracy in event logging.