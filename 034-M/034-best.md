Blunt Butter Moose

medium

# A malicious liquidator could cause the liquidation process to be stuck
## Summary

Malicious liquidators could cause the liquidation process to be stuck as they could trigger Stage 1 (`liquidatePartyA`) of the liquidation process but choose not to proceed with Stage 2 (`setSymbolsPrice`), resulting in the liquidation being stuck as no one else can trigger the Stage 2 (`setSymbolsPrice`) on behalf of the first liquidator.

## Vulnerability Detail

The new update introduces a new process/mechanism for liquidating a PartyA. When a PartyA becomes liquidatable, any liquidator could obtain the 'LiquidationSig' from Muon, and they will be issued a unique `reqId` +`liquidationId`

The fastest liquidator (Liquidator A) that executed the `liquidatePartyA` function (Stage 1) will lock their `liquidationId` into the PartyA's `liquidationDetails` storage, and `setSymbolsPrice` function (Stage 2) will verify the `liquidationId` to ensure that it is the same as the one store in the storage in the previous stage.

Only the person that calls the `liquidatePartyA` (Stage 1) can call `setSymbolsPrice` (Stage 2) because no one else can generate a `LiquidationSig` with a `liquidationId` tagged to someone else.

However, the issue with this design is that a malicious liquidator could trigger Stage 1 (`liquidatePartyA`) of the liquidation process but choose not to proceed with Stage 2 (`setSymbolsPrice`), resulting in the liquidation being stuck as no one else can trigger the Stage 2 (`setSymbolsPrice`) on behalf of the first liquidator.

> **Note**
> This is not an issue in the previous design because the liquidator does not have to be the same for `liquidatePartyA` (Stage 1) and `setSymbolsPrice` (Stage 2).

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L20

```solidity
File: LiquidationFacetImpl.sol
20:     function liquidatePartyA(address partyA, LiquidationSig memory liquidationSig) internal {
..SNIP..
32:         require(availableBalance < 0, "LiquidationFacet: PartyA is solvent");
33:         maLayout.liquidationStatus[partyA] = true;
34:         AccountStorage.layout().liquidationDetails[partyA] = LiquidationDetail({
35:             liquidationId: liquidationSig.liquidationId,
36:             liquidationType: LiquidationType.NONE,
37:             upnl: liquidationSig.upnl,
38:             totalUnrealizedLoss: liquidationSig.totalUnrealizedLoss,
39:             deficit: 0,
40:             liquidationFee: 0,
41:             timestamp: liquidationSig.timestamp
42:         });
43:         AccountStorage.layout().liquidators[partyA].push(msg.sender);
44:     }
```

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L46

```solidity
File: LiquidationFacetImpl.sol
46:     function setSymbolsPrice(address partyA, LiquidationSig memory liquidationSig) internal {
47:         MAStorage.Layout storage maLayout = MAStorage.layout();
48:         AccountStorage.Layout storage accountLayout = AccountStorage.layout();
49: 
50:         LibMuon.verifyLiquidationSig(liquidationSig, partyA);
51:         require(maLayout.liquidationStatus[partyA], "LiquidationFacet: PartyA is solvent");
52:         require(
53:             keccak256(accountLayout.liquidationDetails[partyA].liquidationId) ==
54:                 keccak256(liquidationSig.liquidationId),
55:             "LiquidationFacet: Invalid liqiudationId"
56:         );
```

> **Note**
> The role of the liquidator is not trusted as per the contest's README (https://github.com/sherlock-audit/2023-08-symmetrical-xiaoming9090#q-are-there-any-additional-protocol-roles-if-yes-please-explain-in-detail)
>
> **Q: Are there any additional protocol roles? If yes, please explain in detail:**
>
> MUON_SETTER_ROLE: Can change settings of the Muon Oracle. SYMBOL_MANAGER_ROLE: Can add, edit, and remove markets, as well as change market settings like fees and minimum acceptable position size. PAUSER_ROLE: Can pause all system operations. UNPAUSER_ROLE: Can unpause all system operations. PARTY_B_MANAGER_ROLE: Can add new partyBs to the system. LIQUIDATOR_ROLE: Can liquidate users. SETTER_ROLE: Can change main system settings. **Note: All roles are trusted except for LIQUIDATOR_ROLE.**

## Impact

The liquidation process cannot be completed, leading to a loss of funds. Medium impact as the consequences of this issue is a variation of this [report](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/293) and its duplicates.

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L20

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L46

## Tool used

Manual Review

## Recommendation

If the first liquidator does not complete stage 2 of the liquidation process (`setSymbolsPrice`) within a specific period of time, consider allowing another liquidator to take over to break the DOS.