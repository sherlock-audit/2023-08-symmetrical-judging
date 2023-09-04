Blunt Butter Moose

medium

# Dependency on PartyB's liquidation status
## Summary

The newly added restriction creates some dependencies on the completion status of the PartyB 's liquidation process, which could potentially lead to stuck assets.

## Vulnerability Detail

The updated `liquidatePositionsPartyA` function added a new check below to prevent quotes belonging to a liquidatable PartyB that is currently undergoing the liquidation process from being executed. The mechanism works as follows:

Assume that PartyA trades with the following:

- PartyB1 with Quote1 and Quote2
- PartyB2 with Quote3 and Quote4

At T0, PartyB2 becomes liquidatable, and the liquidation process is ongoing with $Liquidator_B$

At T1, PartyA becomes liquidatable due to losing trades with PartyB1. Thus, PartyA also has an ongoing liquidation. 

$Liquidator_A$ perform the liquidation for PartyA. The liquidation fee will only be released to $Liquidator_A$ when all the PartyA's quotes have been liquidated (`partyAPositionsCount` == 0). $Liquidator_A$ liquidate Quote1 and Quote2 without any issue. 

$Liquidator_A$ has to liquidate the remaining two quotes to complete the liquidation process of PartyA.  However, when $Liquidator_A$ attempts to liquidate Quote3 and Quote4 of PartyB2 via the `liquidatePositionsPartyA` function, it will revert because the remaining quotes belong to the PartyB which also has an ongoing liquidation. Thus, $Liquidator_A$ has to wait until the PartyB's liquidation process is completed.

This creates some dependencies on the completion status of the PartyB 's liquidation process, which could result in the following issues if liquidators do not liquidate a PartyB in a timely manner (maliciously or unintentionally)

- PartyA can only complete its liquidation if PartyB completes its liquidation. PartyA is stuck in a liquidatable state and cannot perform any trades.
- $Liquidator_A$ can only receive its liquidation fee if PartyB completes its liquidation. The liquidator's liquidation fee is stuck.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L138

```diff
function liquidatePositionsPartyA(
    address partyA,
    uint256[] memory quoteIds
) internal returns (bool) {
	..SNIP..
    require(maLayout.liquidationStatus[partyA], "LiquidationFacet: PartyA is solvent");
    for (uint256 index = 0; index < quoteIds.length; index++) {
        Quote storage quote = quoteLayout.quotes[quoteIds[index]];
        require(
            quote.quoteStatus == QuoteStatus.OPENED ||
                quote.quoteStatus == QuoteStatus.CLOSE_PENDING ||
                quote.quoteStatus == QuoteStatus.CANCEL_CLOSE_PENDING,
            "LiquidationFacet: Invalid state"
        );
+        require(
+            !maLayout.partyBLiquidationStatus[quote.partyB][partyA],
+            "LiquidationFacet: PartyB is in liquidation process"
+        );
```

## Impact

Affected PartyA is stuck in a liquidatable state and cannot perform any trades, and the liquidator's liquidation fee is stuck.

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L138

## Tool used

Manual Review

## Recommendation

Consider monitoring for events where the liquidation of PartyB has not been completed within a specific period, and have the protocol to step in to carry out the liquidation of PartyB if this occurs so that liquidation of PartyA can proceed.