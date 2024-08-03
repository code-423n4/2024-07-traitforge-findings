# Low Risk Issues

| ID |Title|
|:--:|:-------|
|[L-01]| Excess ether sent to `EntityForging` not refunded |
|[L-02]| Admin setter functions lack proper validations |
|[L-03]| Lack of incentives for minting low value NFTs can lead to minting getting stuck |

## [L-01] Excess ether sent to `EntityForging` not refunded

The `forgeWithListed` function in `EntityForging.sol` does not refund excess ether sent to the contract. In the event that the user sends more ether than required, the surplus will get stuck in the contract.

## [L-02] Admin setter functions lack proper validations

Setter functions in `NukeFund.sol`, `EntityForging.sol`, `EntityTrading`, `EntropyGenerator` and `TraitForgeNft.sol` do not apply proper validation to ensure that the values being set are within the expected range, which could lead to unexpected behavior in the contracts.

## [L-03] Lack of incentives for minting low value NFTs can lead to minting getting stuck

The entropy values and thus, the potential value of the tokens, are known before minting. As prices are prices are also fixed for a given token, this would make some tokens much more desirable than others. It is acknowledged that the racing for minting valuable tokens is part of the game. However, for low value tokens, there might not be enough incentive for anyone to mint them, taking the burden, and thus, the minting of the NFTs might get stuck. This might be specially problematic at the end of a generation, when the price of the NFTs is much higher than the price of the NFTs that can be minted at the beginning of the generation.


# Non-Critical Issues

| ID |Title|
|:--:|:-------|
|[N-01]| No need to generate initial entropy in batches |
|[N-02]| Incorrect check for slot index out of bounds |
|[N-03]| Incorrect comments and typos |
|[N-04]| Unnecessary checks |

## [N-01] No need to generate initial entropy in batches

Given that the block gas limit in the Base network is 120M, there is not need to generate the initial entropy in batches, being possible to do it in a single transaction, ideally on contract deployment.

## [N-02] Incorrect check for slot index out of bounds

The check in `EntropyGenerator.getEntropy` should be `slotIndex < maxSlotIndex` instead of `slotIndex <= maxSlotIndex`, as a value of 770 will throw an out of bounds error.

```solidity
File: EntropyGenerator.sol

168: require(slotIndex <= maxSlotIndex, 'Slot index out of bounds.');
```

## [N-03] Incorrect comments and typos

```solidity
File: EntityForging.sol

196    emit CancelledListingForForging(tokenId); // Emitting with 0 fee to denote cancellation 👈
```

```solidity
File: EntityTrading.sol
                      👇
037:  // function to lsit NFT for sale
                                                                            👇
053:     nftContract.transferFrom(msg.sender, address(this), tokenId); // trasnfer NFT to contract
                                                         👇
097:    // check if caller is the seller and listing is acivte
                                👇
100:      'Only the seller can canel the listing.'
```

```solidity
File: EntropyGenerator.sol
                                                  👇                                          👇
046:  // Functions to initalize entropy values inbatches to spread gas cost over multiple transcations
                                                                                      👇
130:  // function to get the last initialized index for debugging or informational puroposed
                                                👇       👇
135:  // function to derive various parameters baed on entrtopy values, demonstrating potential cases
             👇
151:    // exmaple logic to determine a boolean property based on entropy
                                                                                                👇
160:    return (nukeFactor, forgePotential, performanceFactor, isForger); // return derived parammeters
                                                👇
184:    return paddedEntropy; // return the caculated entropy value
                                👇
187:  // Utility function te calcualte the number of digits in a number
```

```solidity
File: NukeFund.sol
                        👇
109:  function setAgeMultplier(uint256 _ageMultiplier) external onlyOwner {
                    👇
126:    uint256 perfomanceFactor = nftContract.getTokenEntropy(tokenId) % 10;
               👇
129:      perfomanceFactor *
                                                         👇
145:    uint256 initialNukeFactor = entropy / 40; // calcualte initalNukeFactor based on entropy, 5 digits
```

```solidity
File: TraitForgeNft.sol
                                                                    👇
357:  // distributes funds to nukeFund contract, where its then distrubted 10% dev 90% nukeFund
```

## [N-04] Unnecessary checks

```solidity
File: NukeFund.sol

// @audit `burn` already checks for approval
158:     require(
159:       nftContract.getApproved(tokenId) == address(this) ||
160:         nftContract.isApprovedForAll(msg.sender, address(this)),
161:       'Contract must be approved to transfer the NFT.'
162:     );
```

```solidity
File: EntityTrading.sol

// @audit `transferFrom` already checks for approval
47:     require(
48:       nftContract.getApproved(tokenId) == address(this) ||
49:         nftContract.isApprovedForAll(msg.sender, address(this)),
50:       'Contract must be approved to transfer the NFT.'
51:     );
52: 
53:     nftContract.transferFrom(msg.sender, address(this), tokenId); // trasnfer NFT to contract
```