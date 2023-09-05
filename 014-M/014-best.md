Calm Latte Hamster

medium

# `setCollateral` can be DOS'd by depositing 1 wei of collateral
## Summary

`setCollateral` admin action requires balance of the current collateral to be 0 before changing it to a new collateral. Any user can front-run all such transactions by depositing 1 wei of collateral to make `setCollateral` revert very cheaply.

## Vulnerability Detail

`setCollateral` has the following require:
```solidity
    require(
        IERC20Metadata(GlobalAppStorage.layout().collateral).balanceOf(address(this)) == 0,
        "ControlFacet: There is still collateral in the contract"
    );
```

If any user deposits 1 wei of old collateral just before `setCollateral` is called, then it will be impossible to use this function.

## Impact

Admin is unable to change collateral due to cheap DOS attack.

## Code Snippet

The following check in the `setCollateral` allows to DOS it by depositing 1 wei of collateral:
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/control/ControlFacet.sol#L120-L123

## Tool used

Manual Review

## Recommendation

Consider transferring all collateral to admin or other specified address when switching instead of requiring balance to be 0. Another option is to revert if amount is large enough, but to send to admin and proceed if amount is small.