Blunt Butter Moose

high

# Ineffective liquidation signatures expiration mechanism
## Summary

The new PartyA's liquidation process relies on the timestamp to expire old liquidation signatures, which is ineffective.

## Vulnerability Detail

Assume that `MuonStorage.layout().upnlValidTime` is set to 15 minutes. In this case, the `liquidatePartyA` function accepts the liquidation signature generated up to 15 minutes ago.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L20

```solidity
File: LiquidationFacetImpl.sol
20:     function liquidatePartyA(address partyA, LiquidationSig memory liquidationSig) internal {
21:         MAStorage.Layout storage maLayout = MAStorage.layout();
22: 
23:         LibMuon.verifyLiquidationSig(liquidationSig, partyA);
24:         require(
25:             block.timestamp <= liquidationSig.timestamp + MuonStorage.layout().upnlValidTime,
26:             "LiquidationFacet: Expired signature"
27:         );
```

Assume that Alice account is liquidatable. $Liquidator_A$ and $Liquidator_B$ generate a liquidation signature within the time period highlighted in green below. Subsequently, at T0, $Liquidator_A$ calls the `liquidatePartyA` function to initiate the liquidation process and "lock-in" its liquidation ID.

For simplicity's sake, the liquidation process is executed atomically within the same block. As such, the liquidation process is completed at T0.

The issue is that if Alice started trading again within T0 and T1, she could be liquidated again by $Liquidator_A$ or $Liquidator_B$ using back the old liquidation signature even if her account is healthy.

![](https://user-images.githubusercontent.com/102820284/265329342-282dd92e-d1db-40c7-836a-ed81597a7bf1.png)

> **Note**
> The role of the liquidator is not trusted as per the contest's README (https://github.com/sherlock-audit/2023-08-symmetrical-xiaoming9090#q-are-there-any-additional-protocol-roles-if-yes-please-explain-in-detail)
>
> **Q: Are there any additional protocol roles? If yes, please explain in detail:**
>
> MUON_SETTER_ROLE: Can change settings of the Muon Oracle. SYMBOL_MANAGER_ROLE: Can add, edit, and remove markets, as well as change market settings like fees and minimum acceptable position size. PAUSER_ROLE: Can pause all system operations. UNPAUSER_ROLE: Can unpause all system operations. PARTY_B_MANAGER_ROLE: Can add new partyBs to the system. LIQUIDATOR_ROLE: Can liquidate users. SETTER_ROLE: Can change main system settings. **Note: All roles are trusted except for LIQUIDATOR_ROLE.**

## Impact

Liquidating a healthy account would result in a loss of assets for the victim.

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L20

## Tool used

Manual Review

## Recommendation

The root cause is that the new design relies only on the timestamp to expire old liquidation signatures, which is insufficient to protect against replay attacks.

To adequately guard against this attack, all liquidation signatures generated at or before T0 should be automatically marked as invalid after the liquidation process is completed at T0. In this case, it is no longer possible for $Liquidator_A$ and $Liquidator_B$ using back the old liquidation signature, which was generated at or before T0.

One possible solution is to save the timestamp of the last liquidation completed (`lastLiquidationCompleted`). In our example, once the liquidation process is completed at T0, T0 will be saved to `lastLiquidationCompleted` state. When the liquidator calls the `liquidatePartyA` function to initiate the liquidation process, the function can check if the liquidation signature provided is newer than `lastLiquidationCompleted`, effectively preventing anyone from using the old liquidation signatures from the last liquidation event.

> **Note**
> Some readers might think that nonce would solve this issue. However, as highlighted in many of the issues in the main audit, using back the nonce mechanism would allow PartyA to block liquidators from liquidating their accounts by incrementing the nonce in an attempt to invalidate the liquidator's signature, leading to another issue.