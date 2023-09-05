Calm Latte Hamster

high

# MultiAccount `depositAndAllocateForAccount` function doesn't scale the allocated amount correctly, failing to allocate enough funds
## Summary

This is an issue very similar to [issue 222](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/222) from previous audit contest, but in a `MultiAccount` smart contract.

`MultiAccount.depositAndAllocateForAccount` uses the same `amount` value both for `depositFor` and for `allocate`. However, deposit amount decimals are from the collateral token while allocate amount decimals = 18. This means that for USDC (decimals = 6), `depositAndAllocateForAccount` will deposit correct amount, but allocate amount which is 1e12 times smaller (dust amount).

## Vulnerability Detail

Internal accounting (allocatedBalances) are tracked as fixed numbers with 18 decimals, while collateral tokens can have different amount of decimals. This is correctly accounted for in `AccountFacet.depositAndAllocate`:
```solidity
    AccountFacetImpl.deposit(msg.sender, amount);
    uint256 amountWith18Decimals = (amount * 1e18) /
        (10 ** IERC20Metadata(GlobalAppStorage.layout().collateral).decimals());
    AccountFacetImpl.allocate(amountWith18Decimals);
```

But it is treated incorrectly in `MultiAccount.depositAndAllocateForAccount`:
```solidity
    ISymmio(symmioAddress).depositFor(account, amount);
    bytes memory _callData = abi.encodeWithSignature(
        "allocate(uint256)",
        amount
    );
    innerCall(account, _callData);
```

This leads to incorrect allocated amounts.

## Impact

Similar to 222 from previous audit contest, the user expects to have full amount deposited and allocated, but ends up with only dust amount allocated, which can lead to unexpected liquidations (for example, user is at the edge of liquidation, calls depositAndAllocate to improve account health, but is liquidated instead). For consistency reasons, since this is almost identical to 222, it should also be high.

## Code Snippet

The same amount is used for `depositFor` and `allocate`:
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/multiAccount/MultiAccount.sol#L167-L173

## Tool used

Manual Review

## Recommendation

Scale amount correctly before allocating it:
```solidity
    ISymmio(symmioAddress).depositFor(account, amount);
+   uint256 amountWith18Decimals = (amount * 1e18) /
+       (10 ** IERC20Metadata(collateral).decimals());
    bytes memory _callData = abi.encodeWithSignature(
        "allocate(uint256)",
-       amount
+       amountWith18Decimals
    );
    innerCall(account, _callData);
```
