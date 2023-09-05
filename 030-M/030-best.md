Blunt Butter Moose

medium

# Certain quotes might not be able to close
## Summary

Certain quotes might not be able to close due to the new require statements in the updated codebase.

## Vulnerability Detail

The protocol team fixes [Issue 251](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/251) by adding 3 require statements at Lines 138 to 140 below to ensure that it does not round down to zero.

However, if the remaining `filledAmount` of a position happens to be too small, it will inevitably lead to the require statement to always revert, leading to the position being unable to close.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibQuote.sol#L134

```solidity
File: LibQuote.sol
134:     function closeQuote(Quote storage quote, uint256 filledAmount, uint256 closedPrice) internal {
135:         QuoteStorage.Layout storage quoteLayout = QuoteStorage.layout();
136:         AccountStorage.Layout storage accountLayout = AccountStorage.layout();
137:         SymbolStorage.Layout storage symbolLayout = SymbolStorage.layout();
138: @>      require(quote.lockedValues.cva * filledAmount / LibQuote.quoteOpenAmount(quote) > 0, "LibQuote: Low filled amount");
139: @>      require(quote.lockedValues.mm * filledAmount / LibQuote.quoteOpenAmount(quote) > 0, "LibQuote: Low filled amount");
140: @>      require(quote.lockedValues.lf * filledAmount / LibQuote.quoteOpenAmount(quote) > 0, "LibQuote: Low filled amount");
141:         LockedValues memory lockedValues = LockedValues(
142:             quote.lockedValues.cva -
143:             ((quote.lockedValues.cva * filledAmount) / (LibQuote.quoteOpenAmount(quote))),
144:             quote.lockedValues.mm -
145:             ((quote.lockedValues.mm * filledAmount) / (LibQuote.quoteOpenAmount(quote))),
146:             quote.lockedValues.lf -
147:             ((quote.lockedValues.lf * filledAmount) / (LibQuote.quoteOpenAmount(quote)))
148:         );
149:         accountLayout.lockedBalances[quote.partyA].subQuote(quote).add(lockedValues);
150:         accountLayout.partyBLockedBalances[quote.partyB][quote.partyA].subQuote(quote).add(
151:             lockedValues
152:         );
153:         quote.lockedValues = lockedValues;
```

## Impact

Certain quotes might not be able to close and will be stuck.

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibQuote.sol#L134

## Tool used

Manual Review

## Recommendation

Consider providing the users an option to work around by closing the entire position completely. If a position is closed completely, there is no need to adjust the locked values, which overcomes potential DOS.

```diff
function closeQuote(Quote storage quote, uint256 filledAmount, uint256 closedPrice) internal {
    QuoteStorage.Layout storage quoteLayout = QuoteStorage.layout();
    AccountStorage.Layout storage accountLayout = AccountStorage.layout();
    SymbolStorage.Layout storage symbolLayout = SymbolStorage.layout();
    
+	LockedValues memory lockedValue;
+   if (LibQuote.quoteOpenAmount(quote) == quote.quantityToClose) {
+		lockedValue = quote.lockedValues;
+   } else {
        require(quote.lockedValues.cva * filledAmount / LibQuote.quoteOpenAmount(quote) > 0, "LibQuote: Low filled amount");
        require(quote.lockedValues.mm * filledAmount / LibQuote.quoteOpenAmount(quote) > 0, "LibQuote: Low filled amount");
        require(quote.lockedValues.lf * filledAmount / LibQuote.quoteOpenAmount(quote) > 0, "LibQuote: Low filled amount");
-       LockedValues memory lockedValues = LockedValues(
+       lockedValues = LockedValues(
            quote.lockedValues.cva -
            ((quote.lockedValues.cva * filledAmount) / (LibQuote.quoteOpenAmount(quote))),
            quote.lockedValues.mm -
            ((quote.lockedValues.mm * filledAmount) / (LibQuote.quoteOpenAmount(quote))),
            quote.lockedValues.lf -
            ((quote.lockedValues.lf * filledAmount) / (LibQuote.quoteOpenAmount(quote)))
        );
+	}
    accountLayout.lockedBalances[quote.partyA].subQuote(quote).add(lockedValues);
    accountLayout.partyBLockedBalances[quote.partyB][quote.partyA].subQuote(quote).add(
        lockedValues
    );
    quote.lockedValues = lockedValues;
```