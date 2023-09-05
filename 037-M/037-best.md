Blunt Butter Moose

medium

# Some actions are allowed on partyB when corresponding partyA is liquidated
## Summary

Some actions are still allowed on partyB when the corresponding partyA is liquidated. Thus, the [Issue 189](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/189) was deemed not sufficiently remediated.

## Vulnerability Detail

This report specifically addresses the [Issue 189](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/189). Referring to the attack path mentioned in Issue 189, the attacker would need to withdraw their ill-gotten gains from the protocol after Step 7. There are two ways to do so, as mentioned in the original report:

1. Via `deallocateForPartyB` function as it doesn't check if partyA is liquidated, allowing partyB to deallocate funds when partyA liquidation process is not finished
2. Via `transferAllocation` function as it doesn't check if partyA is liquidated. The attacker could transfer the allocation to another account and withdraw from there.

During the fix review, it was found that the `deallocateForPartyB` has been fixed by adding the `notLiquidatedPartyA` modifier to prevent it from being executed while PartyA is in liquidatable state.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/Account/AccountFacet.sol#L74

```diff
function deallocateForPartyB(
    uint256 amount,
    address partyA,
    SingleUpnlSig memory upnlSig
- ) external whenNotPartyBActionsPaused notLiquidatedPartyB(msg.sender, partyA) onlyPartyB {
+ ) external whenNotPartyBActionsPaused notLiquidatedPartyB(msg.sender, partyA) notLiquidatedPartyA(partyA) onlyPartyB {
    AccountFacetImpl.deallocateForPartyB(amount, partyA, upnlSig);
    emit DeallocateForPartyB(msg.sender, partyA, amount);
}
```

The fix is also applied to `transferAllocation` function to ensure that PartyA and PartyB are not in a liquidatable state.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/Account/AccountFacetImpl.sol#L70

```diff
function transferAllocation(
    uint256 amount,
    address origin,
    address recipient,
    SingleUpnlSig memory upnlSig
) internal {
    MAStorage.Layout storage maLayout = MAStorage.layout();
    AccountStorage.Layout storage accountLayout = AccountStorage.layout();
    require(
        !maLayout.partyBLiquidationStatus[msg.sender][origin],
        "PartyBFacet: PartyB isn't solvent"
    );
    require(
        !maLayout.partyBLiquidationStatus[msg.sender][recipient],
        "PartyBFacet: PartyB isn't solvent"
    );
+    require(
+        !MAStorage.layout().liquidationStatus[origin],
+        "PartyBFacet: Origin isn't solvent"
+    );
+    require(
+        !MAStorage.layout().liquidationStatus[recipient],
+        "PartyBFacet: Recipient isn't solvent"
+    );
```

In addition, the recommendation section of the original report also highlighted the following two functions that need to be fixed:

- `requestToClosePosition`
- `fillCloseRequest`

During the main audit, the above two functions are deemed to allow malicious users to move assets/balances from one account to the other, which can potentially be exploited to withdraw ill-gotten gains out of the protocol. Thus, they should be adequately guarded.

However, the fix is not applied to these two functions. Thus, the [Issue 189](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/189) was deemed not sufficiently remediated.

## Impact

Medium as the impact is the same as the original report.

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/PartyA/PartyAFacet.sol#L85

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacet.sol#L137

## Tool used

Manual Review

## Recommendation

As per the recommendation in the [original report](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/189), add require's (or modifiers) to check that neither partyA nor partyB of the quote are liquidated in the following functions:

1. `requestToClosePosition`
2. `fillCloseRequest`