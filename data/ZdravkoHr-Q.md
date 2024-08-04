# [L-01] `fetchListings` returns an additional listing
[EntityForging.fetchListings](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L48-L53) returns all existing listings plus an additional empty one at the front of the array. 

```solidity
    _listings = new Listing[](listingCount + 1);
    for (uint256 i = 1; i <= listingCount; ++i) {
      _listings[i] = listings[i];
    }
```

This is inaccurate and may cause problems with integrations. Make the following changes: 

```diff
-   _listings = new Listing[](listingCount + 1);
+   _listings = new Listing[](listingCount);
 
-   for (uint256 i = 1; i <= listingCount; ++i) {
+   for (uint256 i = 0; i < listingCount; ++i) {
-     _listings[i] = listings[i];
+     _listings[i] = listings[i + 1];
    }
```

# [L-02] Some listed forgings cannot be forged
If the forger token is older than the previous generation, it cannot be forged, but listing of such tokens is allowed through [EntityForging.listForForging()](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L67-L100). 

# [L-03] EntityForging and EntityTrading entites can be cancelled immediately
In both contracts users may create a listing and cancel it immediately. A malicious user can leverage this to create a really big number of listings and DoS [EntityForging.fetchListings](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L48-L53) by causing an OOG exception. 

Consider adding a small fee when canceling.

# [L-04] Canceling a listing doesn't update all mappings
[EntityForging._cancelListingForForging] and [EntityTrading.cancelListing](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L94-L108) don't delete the value of the `listedTokenIds` mapping. 
In result the mapping will not be a reliable source of information for `token - listing` relationship.

# [L-05] DevFund.claim can be called for empty devInfo

[DevFund.claim()](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/DevFund/DevFund.sol#L61C1-L75C4) can be called on an empty entry because there is no validation for its existance.

# [L-06] Forger owner can grief users that forge

When the payment on forge is [executed](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L158-L159), the forger owner may revert the transaction by reverting in its receive function and causing loss of funds (gas price) for the forging user.

# [L-07] Zero entropy tokens can be burned
[NukeFund.burn()](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153-L182) will not give any rewards for burning a token with 0 entropy, but the token will still be burned.

Consider adding a revert if the entropy is 0.