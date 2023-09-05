Calm Latte Hamster

medium

# Wrong calculation of solvency in `fillCloseRequest` prevents the position from being closed even if the user is solvent after position closure
## Summary

This is an [issue 184](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/184) from previous contest, developers fixed it only partially (request to close), but fill close request remains the same.
If partyA has requested to close position, and partyB has called `fillCloseRequest`, the transaction will revert even though partyA is solvent after closure in the following scenario:
- partyA is not solvent if the position is closed at the current market price
- `closePrice` provided by the partyB is better than market price
- partyA is solvent if the position is closed at the `closePrice` provided by partyB

This is unfair for partyA and can lead to liquidation and loss of funds for partyA, even though it could have had the position closed correctly to make it solvent.

## Vulnerability Detail

`isSolventAfterClosePosition` verifies that both partyA and partyB are solvent at the market price, then verifies that both partyA and partyB are solvent at the `closePrice`. However, if both parties are solvent at `closePrice` - this should be sufficient, as it's the most fair way for both parties - let the position close if both parties are solvent after it. But the requirement to be solvent both at market and close price prevents the transaction from being completed successfully.

## Impact

PartyA can be liquidated and lose funds instead of correct closure of the position due to failed position closure transaction.

## Code Snippet

`isSolventAfterClosePosition` requires enough balance both at the market and close price:
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibSolvency.sol#L101-L133

## Tool used

Manual Review

## Recommendation

Require both parties to only be solvent at `closePrice` when the position is closed, there is no point in reverting if the position closure can make both parties solvent, even though one of it is not solvent before that.