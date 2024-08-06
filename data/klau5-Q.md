# Unable to pause contracts

## Links to affected code

[https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/DevFund/DevFund.sol#L9](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/DevFund/DevFund.sol#L9)

[https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L10](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L10)

[https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L11](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L11)

[https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L9](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L9)

[https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L11](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L11)

[https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L19](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L19)

## Impact

Cannot pause the contract in an emergency situation.

## Proof of Concept

The DevFund, EntityTrading, Airdrop, EntityForging, EntropyGenerator, NukeFund, and TraitForgeNft contracts inherit Pausable and use the `whenNotPaused` modifier. However, they do not have a public function that calls `Pausable._pause`. Therefore, it is practically impossible to pause these contracts.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Create a public function in the contracts using Pausable that calls `Pausable._pause` so that the administrator can pause the contract.

# Cannot transfer developer’s share to the DAO in NukeFund

## Links to affected code

[https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L50-L54](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L50-L54)

[https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/Airdrop/Airdrop.sol#L38](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/Airdrop/Airdrop.sol#L38)

## Impact

Cannot transfer developer’s share to the DAO in NukeFund

## Proof of Concept

The NukeFund contract is responsible for distributing the native tokens it receives. If `Airdrop.daoAllowed` is set in the Airdrop contract, the developer's share is transferred to the DAO. If `Airdrop.daoAllowed` is not set, the tokens are transferred to the contract admin.

```solidity
  receive() external payable {
    uint256 devShare = msg.value / taxCut; // Calculate developer's share (10%)
    uint256 remainingFund = msg.value - devShare; // Calculate remaining funds to add to the fund

    fund += remainingFund; // Update the fund balance

@>  if (!airdropContract.airdropStarted()) {
      (bool success, ) = devAddress.call{ value: devShare }('');
      require(success, 'ETH send failed');
      emit DevShareDistributed(devShare);
@>  } else if (!airdropContract.daoFundAllowed()) {
      (bool success, ) = payable(owner()).call{ value: devShare }('');
      require(success, 'ETH send failed');
    } else {
@>    (bool success, ) = daoAddress.call{ value: devShare }('');
      require(success, 'ETH send failed');
      emit DevShareDistributed(devShare);
    }
    ...
  }
```

To set `Airdrop.daoAllowed`, the `Airdrop.allowDaoFund` function must be called. This function can only be called by the admin of the Airdrop contract. The admin of the Airdrop contract is the TraitForgeNft contract, which does not have a function to call `Airdrop.allowDaoFund`. The deployment script also does not call `Airdrop.allowDaoFund`.

```solidity
@>function allowDaoFund() external onlyOwner {
    require(started, 'Not started');
    require(!daoAllowed, 'Already allowed');
    daoAllowed = true;
  }
```

Therefore, `Airdrop.daoAllowed` cannot be set, and the feature to transfer the developer's share to the DAO in the NukeFund contract's receive function cannot be executed.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Create a public function in the TraitForgeNft contract to call `Airdrop.allowDaoFund` and allow the admin to call it.