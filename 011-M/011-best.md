Calm Latte Hamster

medium

# Liquidation of partyB may get stuck if muon app is not available (down) for some periods of time.
## Summary

PartyB liquidation is a 2-step process: initiating liquidaiton and liquidating positions (possibly in multiple transactions). In order to perform the 2nd step, the liquidator has to obtain quotes signature from muon app with a timestamp in the liquidation timeout window after the liquidation timestamp. If muon app is not available during this time, or if all liquidators are offline for any reason, the liquidation of partyB will be stuck and never finish, causing loss of funds (funds stuck forever) for corresponding partyA, because locked balances will keep quotes with this partyB forever and prevent partyA from withdrawing all funds owed to it. Also, if partyA is later liquidated, liquidation for partyA won't be able to finish, because it won't be able to liquidate quotes with this partyB (since it's liquidated).

## Vulnerability Detail

When partyB calls `liquidatePositionsPartyB`, it has to supply QuotePriceSig with a list of quotes to liquidate. This signature timestamp is required to be:
```solidity
    require(
        priceSig.timestamp <=
            maLayout.partyBLiquidationTimestamp[partyB][partyA] + maLayout.liquidationTimeout,
        "LiquidationFacet: Invalid signature"
    );
    ...
    require(
        maLayout.partyBLiquidationTimestamp[partyB][partyA] <= priceSig.timestamp,
        "LiquidationFacet: Expired signature"
    );
```

Signature timestamp must be between liquidation timestamp of partyB and liquidation timestamp + liquidation timeout.
If it's not possible to obtain such a signature for whatever reason, it will be impossible to finish the liquidation.

## Impact

PartyA will have lockedBalances stuck forever with liquidated partyB, preventing it from withdrawing all funds. If partyA is ever liquidated after that, its liquidation won't be able to be finished either, because liquidation process won't be able to liquidate quotes between these parties due to partyB being in the liquidation status.

## Code Snippet

`liquidatePositionsPartyB` has the following check for the signature timestamp:
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L327-L339

## Tool used

Manual Review

## Recommendation

Consider removing the liquidation timeout, because the price information in this function is only used for informational purpose (emitted event) and not in any impactful calculations. As it is now, the prices in `liquidatePositionsPartyB` can differ from prices at liquidation initiation anyway and pnl calculated from these prices might indicate that partyB was solvent. So there won't be much of a problem if this limit is removed.