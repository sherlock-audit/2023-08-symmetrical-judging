Calm Latte Hamster

medium

# Position value can fall below minimum acceptable quote value when partially closing positions requested to be closed in full
## Summary

This is [issue 248](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/248) from previous audit contest, which was fixed incorrectly.
When PartyA requests to close LIMIT position in full, but partyB closes it partially, the remaining open quote can be below `minAcceptableQuoteValue`, breaking important protocol invariant, which can cause different problems, such as not enough incentive to liquidate dust positions.

## Vulnerability Detail

In `LibQuote.closeQuote` there is a requirement to have the remaining quote value to not be less than `minAcceptableQuoteValue`:
```solidity
if (LibQuote.quoteOpenAmount(quote) != quote.quantityToClose) {
    require(quote.lockedValues.total() >= symbolLayout.symbols[quote.symbolId].minAcceptableQuoteValue,
        "LibQuote: Remaining quote value is low");
}
```

Notice the condition when this require happens:
- `LibQuote.quoteOpenAmount(quote)` is remaining open amount
- `quote.quantityToClose` is requested amount to close

This means that this check is ignored if partyA has requested to close amount equal to full remaining quote value, but enforced when it's not (even if closing fully). For example, a quote with opened amount = 100 is requested to be closed in full (amount = 100): this check is ignored. But PartyB can fill the request partially, for example fill 99 out of 100, and the remainder (1) is not checked to confirm to `minAcceptableQuoteValue`.

The following execution paths are possible if PartyA has open position size = 100 and `minAcceptableQuoteValue` = 5:
- `requestToClosePosition(99)` -> revert
- `requestToClosePosition(100)` -> `fillCloseRequest(99)` -> pass (remaining quote = 1)

## Impact

There can be multiple reasons why the protocol enforces `minAcceptableQuoteValue`, one of them might be the efficiency of the liquidation mechanism: when quote value is too small (and liquidation value too small too), liquidators will not have enough incentive to liquidate these positions in case they become insolvent. Both partyA and partyB might also not have enough incentive to close or respond to request to close such small positions, possibly resulting in a loss of funds and greater market risk for either user.

## Proof of Concept

Add this to any test, for example to `ClosePosition.behavior.ts`.

```solidity
it("Close position with remainder below minAcceptableQuoteValue", async function () {
  const context: RunContext = this.context;

  this.user_allocated = decimal(1000);
  this.hedger_allocated = decimal(1000);

  this.user = new User(this.context, this.context.signers.user);
  await this.user.setup();
  await this.user.setBalances(this.user_allocated, this.user_allocated, this.user_allocated);

  this.hedger = new Hedger(this.context, this.context.signers.hedger);
  await this.hedger.setup();
  await this.hedger.setBalances(this.hedger_allocated, this.hedger_allocated);

  await this.user.sendQuote(limitQuoteRequestBuilder()
    .quantity(decimal(100))
    .price(decimal(1))
    .cva(decimal(10)).lf(decimal(5)).mm(decimal(15))
    .build()
  );
  await this.hedger.lockQuote(1, 0, decimal(5, 17));
  await this.hedger.openPosition(1, limitOpenRequestBuilder().filledAmount(decimal(100)).openPrice(decimal(1)).price(decimal(1)).build());

  // now try to close full position (100)
  await this.user.requestToClosePosition(
    1,
    limitCloseRequestBuilder().quantityToClose(decimal(100)).closePrice(decimal(1)).build(),
  );

  // now partyA cancels request
  //await this.user.requestToCancelCloseRequest(1);

  // partyB can fill 99
  await this.hedger.fillCloseRequest(
    1,
    limitFillCloseRequestBuilder()
      .filledAmount(decimal(99))
      .closedPrice(decimal(1))
      .build(),
  );

  var q = await context.viewFacet.getQuote(1);
  console.log("quote quantity: " + q.quantity.div(decimal(1)) + " closed: " + q.closedAmount.div(decimal(1)));

});
```

Console execution result:
```js
quote quantity: 100 closed: 99
```

## Code Snippet

Notice the condition to perform the `minAcceptableQuoteValue` check:
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibQuote.sol#L155-L158

## Tool used

Manual Review

## Recommendation

The condition should be to ignore the `minAcceptableQuoteValue` if request is filled in full (filledAmount == quantityToClose):
```solidity
-       if (LibQuote.quoteOpenAmount(quote) != quote.quantityToClose) {
+       if (filledAmount != quote.quantityToClose) {
            require(quote.lockedValues.total() >= symbolLayout.symbols[quote.symbolId].minAcceptableQuoteValue,
                "LibQuote: Remaining quote value is low");
        }
```