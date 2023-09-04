Blunt Butter Moose

medium

# PartyB can leverage emergency mode for quick profits
## Summary

PartyB can leverage emergency mode for quick profits via front-running.

## Vulnerability Detail

The updated codebase was found to be vulnerable to the issue highlighted in this report (https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/192).

It was observed that the fixes were to add additional controls at Line 90 below within the `openPosition` function to prevent any PartyB granted with emergency mode to open position.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L90

```solidity
File: PartyBFacetImpl.sol
74:     function openPosition(
75:         uint256 quoteId,
76:         uint256 filledAmount,
77:         uint256 openedPrice,
78:         PairUpnlAndPriceSig memory upnlSig
79:     ) internal returns (uint256 currentId) {
80:         QuoteStorage.Layout storage quoteLayout = QuoteStorage.layout();
81:         AccountStorage.Layout storage accountLayout = AccountStorage.layout();
82: 
83:         Quote storage quote = quoteLayout.quotes[quoteId];
84:         require(accountLayout.suspendedAddresses[quote.partyA] == false, "PartyBFacet: PartyA is suspended");
85:         require(
86:             SymbolStorage.layout().symbols[quote.symbolId].isValid,
87:             "PartyBFacet: Symbol is not valid"
88:         );
89: 
90:         require(!GlobalAppStorage.layout().partyBEmergencyStatus[quote.partyB], "PartyBFacet: PartyB is in emergency mode");
```

However, the newly implemented control can be bypassed via front-running. The attack path is more or less the same.

Bob, the malicious PartyB, sees that the emergency mode is granted to them in the mempool. He front-run the TX and opened any profitable positions as mentioned in the original reports. Once Bob is granted emergency mode, he could close them for a profit.

## Impact

Medium impact as per the original [report](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/192).

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L90

## Tool used

Manual Review

## Recommendation

Consider granting emergency mode to close a selected group of PartyA instead of all PartyA as mentioned in the recommendation of the original report.