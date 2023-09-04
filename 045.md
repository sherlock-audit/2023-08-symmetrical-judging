Damp Flaxen Pelican

medium

# PartyBFacetImpl.chargeFundingRate will never be called with a negative rates array, causing partyA to always pay the funding rate
## Summary

[[PartyBFacetImpl.chargeFundingRate](https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L310-L315)](https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L310-L315) is used to charge funding rate. The value of each element in the `rates` array determines who pays the funding rate. If it is a positive value, then the funding rate will be paid by partyA to partyB. If it is a negative value, vice versa. However, this function can only be called by PartyB, and the `rates` argument is passed in by the caller. This means that partyB can always pass in positive values to charge the funding rate from partyA. Obviously, this is unfair to partyA, which can be regarded as a loss of funds for partyA.

## Vulnerability Detail

```solidity
File: symmio-core\contracts\facets\PartyB\PartyBFacetImpl.sol
310:     function chargeFundingRate(
311:         address partyA,
312:         uint256[] memory quoteIds,
313:         int256[] memory rates,
314:         PairUpnlSig memory upnlSig
315:     ) internal {
316:->       LibMuon.verifyPairUpnl(upnlSig, msg.sender, partyA);
......
329:         for (uint256 i = 0; i < quoteIds.length; i++) {
330:             Quote storage quote = QuoteStorage.layout().quotes[quoteIds[i]];
331:             require(quote.partyA == partyA, "PartyBFacet: Invalid quote");
332:->           require(quote.partyB == msg.sender, "PartyBFacet: Sender isn't partyB of quote");
......
362:             if (rates[i] >= 0) {
363:                 require(
364:                     uint256(rates[i]) <= quote.maxFundingRate,
365:                     "PartyBFacet: High funding rate"
366:                 );
367:                 uint256 priceDiff = (quote.openedPrice * uint256(rates[i])) / 1e18;
368:                 if (quote.positionType == PositionType.LONG) {
369:                     quote.openedPrice += priceDiff;
370:                 } else {
371:                     quote.openedPrice -= priceDiff;
372:                 }
373:->               partyAAvailableBalance -= int256(LibQuote.quoteOpenAmount(quote) * priceDiff / 1e18);
374:->               partyBAvailableBalance += int256(LibQuote.quoteOpenAmount(quote) * priceDiff / 1e18);
375:             } else {
376:                 require(
377:                     uint256(-rates[i]) <= quote.maxFundingRate,
378:                     "PartyBFacet: High funding rate"
379:                 );
380:                 uint256 priceDiff = (quote.openedPrice * uint256(-rates[i])) / 1e18;
381:                 if (quote.positionType == PositionType.LONG) {
382:                     quote.openedPrice -= priceDiff;
383:                 } else {
384:                     quote.openedPrice += priceDiff;
385:                 }
386:->               partyAAvailableBalance += int256(LibQuote.quoteOpenAmount(quote) * priceDiff / 1e18);
387:->               partyBAvailableBalance -= int256(LibQuote.quoteOpenAmount(quote) * priceDiff / 1e18);
388:             }
389:             quote.lastFundingPaymentTimestamp = paidTimestamp;
390:         }
391:         require(partyAAvailableBalance >= 0, "PartyBFacet: PartyA will be insolvent");
392:         require(partyBAvailableBalance >= 0, "PartyBFacet: PartyB will be insolvent");
393:         AccountStorage.layout().partyBNonces[msg.sender][partyA] += 1;
394:         AccountStorage.layout().partyANonces[partyA] += 1;
395:     }
```

L316/L332, determines that o**nly partyB can call this function**.

L332-362, just check whether the quote reaches the time to charge the funding rate.

L363-374, when `rates[i] >= 0`, this code block is executed.

1.  If `quote.positionType == PositionType.LONG`, increasing `quote.openedPrice` means reducing partyA's profit.
2.  If `quote.positionType == PositionType.SHORT`, reducing `quote.openedPrice` means reducing partyA's profit.

Therefore, as long as `rates[i]` is not negative value, `quote.openedPrice` will be updated to price that goes against partyA. This means that partyA pays the funding fee to partyB.

L376-387, when `rates[i] < 0`, this code block is executed. The logic here is opposite to L363-374: partyB pays the funding fee to partyA.

As mentioned above, it is impossible for partyB to allow himself to pay the funding rate to partyA because he can control the `rates` array. Of course, we cannot ignore `quote.maxFundingRate`, which is specified by partyA when creating a quote. If it is 0, then no funding fee will be charged. This report discusses the case where `quote.maxFundingRate > 0`.

## Impact

When `quote.maxFundingRate > 0`, partyB can make partyA always pay the funding rate. This causes partyA to keep losing the funding rate.

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L363-L374

## Tool used

Manual Review

## Recommendation