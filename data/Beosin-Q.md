In the forgeWithListed function of the EntityForging contract, users are required to pay a fee that is not less than the forgingFee. However, if a user overpays the fee, the contract does not return the excess funds to the user, which can result in the user's funds being unnecessarily locked within the contract.

......

    uint256 forgingFee = _forgerListingInfo.fee;
    require(msg.value >= forgingFee, 'Insufficient fee for forging');

    _resetForgingCountIfNeeded(forgerTokenId); // Reset for forger if needed
    _resetForgingCountIfNeeded(mergerTokenId); // Reset for merger if needed

......