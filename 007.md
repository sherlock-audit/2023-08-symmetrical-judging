Calm Latte Hamster

medium

# PartyA fees for pending positions are added to partyA balance after liquidation, which is unfair to partyB or liquidator for LATE or OVERDUE liquidations
## Summary

During PartyA liquidation, pending position fees are returned back to partyA. However, this is done after liquidation. This is unfair to liquidators in case of LATE liquidation and unfair to partyB and/or liquidators in case of OVERDUE liquidation, because pending positions (and fees) are canceled before the actual liquidation of positions, and as such the fees should be added to partyA balance **before** the distribution of profit/cva/lf, not after.

## Vulnerability Detail

The partyA liquidation process takes 4 steps: initiate liquidation, set symbols prices, liquidate pending positions, liquidate active positions.
Since pending positions are canceled, the fee is returned back to partyA and should be used to compensate any LATE or OVERDUE liquidations to make it fair for all parties.

## Impact

Loss of funds for liquidator and/or partyB during LATE or OVERDUE liquidation of PartyA with pending positions.

## Code Snippet

The partyAReimbursement is calculated when liquidating pending positions:
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L113

But applied to partyA balance only after liquidation is finished:
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L221-L223

## Tool used

Manual Review

## Recommendation

Apply reimbursed fees before liquidating active positions (including calculation of liquidation type and possible deficit), not after.