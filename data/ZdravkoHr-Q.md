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