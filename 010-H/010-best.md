Calm Latte Hamster

high

# Position closure might always revert in some cases due to allocatedBalances being unsigned, preventing the user from closing its positions
## Summary

This is [issue 193](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/193) from the previous audit contest. The liquidation part was fixed, but the main reason described here is still the same.

Allocated balances are stored as unsigned ints. 

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/storages/AccountStorage.sol#L37

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/storages/AccountStorage.sol#L41

Total account value is calculated as `allocated balance + unrealized pnl`, for example for liquidation purposes it's calculated as:

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibAccount.sol#L83-L85

In the other cases formula is similar. This means that in case of high unrealized profit, allocated balance might be valid to be negative. For example, a simple case is if account has opened position with unrealized profit of 1000 and locked balance of 100, allocated balance can theoretically be `-900` with account being safe from liquidation: `-900 + 1000 = 100 >= lockedBalance(100)`. But since balance is unsigned, transaction to deallocate such balance will revert. This might be expected behaviour.

However, there are more complex cases possible where unsigned allocated balance can unexpectedly revert different transactions and block users from doing important actions. For example, if user has 2 opposite positions opened of the same quantity (LONG and SHORT) and price has moved significantly, total user `upnl = 0` (because opposite positions hedge each other completely). But the user will be unable to close either of these positions, because closing any position will subract high loss from the position from either partyA or partyB, which will make allocated balance negative and revert the transaction:

Before closing position: 

`Account Balance = Position 1 UPNL (+1000) + Position 2 UPNL (-1000) + Allocated Balance (100) = +100` (account solvent)

After closing position: 

`Account Balance should be = Position 1 UPNL (+1000) + Allocated Balance (-900) = +100` (account solvent), but it will revert.

As such, both partyA and partyB will lose funds deposited into the protocol to keep these positions alive. It will also be impossible to finish liquidation of such positions if any account becomes liquidatable, effectively locking the funds for extended time (until price is in the range where positions can be closed without big loss).

## Vulnerability Detail

The scenario when user unexpectedly can't close position and loses funds:

1. Open 2 large opposite positions between partyA and partyB such that one of them is in a high loss and the other in the same profit. Use minimal allocated balance both for partyA and partyB (because such opposite positions can't be liquidated by themselves as they perfectly hedge each other).
2. When price moves significantly, try to close either of these positions, both will revert.
3. Both partyA and partyB are stuck with the funds in the protocol which they can't take out until price goes back close to initial value.

There might be the other cases with unexpected reverts and users being unable to close their positions.

## Impact

PartyA and PartyB funds can be unexpectedly locked in the protocol for extended time. PartyA or PartyB can allocate additional funds to be able to close positions, then deallocate and withdraw, however depending on situation, required funds to close such position might be orders of magnitude higher than what parties have or are willing to risk (due to leverage).

This is High severity, because the situation can happen by itself for users having multiple positions: naturally some of them will be in profit, some in loss, and total upnl can be much smaller than individual positions, allowing a low allocated balance and making it impossible to close some positions.

## Code Snippet

Add this to any test, for example to `ClosePosition.behavior.ts`.

```ts
it("Unable to close positions due to unsigned allocatedBalances", async function () {
  const context: RunContext = this.context;

  this.user_allocated = decimal(82);
  this.hedger_allocated = decimal(80);

  this.user = new User(this.context, this.context.signers.user);
  await this.user.setup();
  await this.user.setBalances(this.user_allocated, this.user_allocated, this.user_allocated);

  this.hedger = new Hedger(this.context, this.context.signers.hedger);
  await this.hedger.setup();
  await this.hedger.setBalances(this.hedger_allocated, this.hedger_allocated);

  // open 2 opposite direction positions 
  await this.user.sendQuote(limitQuoteRequestBuilder()
    .quantity(decimal(100))
    .price(decimal(1))
    .cva(decimal(10)).lf(decimal(5)).mm(decimal(15))
    .build()
  );
  await this.hedger.lockQuote(1, 0, decimal(4, 17));
  await this.hedger.openPosition(1, limitOpenRequestBuilder().filledAmount(decimal(100)).openPrice(decimal(1)).price(decimal(1)).build());

  await this.user.sendQuote(limitQuoteRequestBuilder()
    .positionType(PositionType.SHORT)
    .quantity(decimal(100))
    .price(decimal(1))
    .cva(decimal(10)).lf(decimal(5)).mm(decimal(15))
    .build()
  );
  await this.hedger.lockQuote(2, 0, decimal(4, 17));
  await this.hedger.openPosition(2, limitOpenRequestBuilder().filledAmount(decimal(100)).openPrice(decimal(1)).price(decimal(1)).build());

  var info = await this.user.getBalanceInfo();
  console.log("partyA allocated: " + info.allocatedBalances / 1e18 + " locked: " + info.totalLocked/1e18 + " pendingLocked: " + info.totalPendingLocked / 1e18);
  var info = await this.hedger.getBalanceInfo(this.user.getAddress());
  console.log("partyB allocated: " + info.allocatedBalances / 1e18 + " locked: " + info.totalLocked/1e18 + " pendingLocked: " + info.totalPendingLocked / 1e18);

  // now the price doubles, with upnl still 0
  // try to close LONG position

  await this.user.requestToClosePosition(
    1,
    limitCloseRequestBuilder().quantityToClose(decimal(100)).closePrice(decimal(2)).build(),
  );

  await expect(this.hedger.fillCloseRequest(
    1,
    limitFillCloseRequestBuilder()
      .filledAmount(decimal(100))
      .closedPrice(decimal(2))
      .build(),
  )).to.be.revertedWith("LibSolvency: PartyB will be liquidatable");
  
  console.log("Attempt to close LONG position failed due to partyB being liquidatable");

  // try to close SHORT position
  await expect(this.user.requestToClosePosition(
    2,
    limitCloseRequestBuilder().quantityToClose(decimal(100)).closePrice(decimal(2)).build(),
    )).to.be.revertedWith("LibSolvency: partyA will be liquidatable");

  console.log("Attempt to close SHORT position failed due to partyA being liquidatable");

});
```

## Tool used

Manual Review

## Recommendation

Make allocated balances signed ints and change all appropriate parts where allocated balances are changed or verified. This will be more user-friendly (less limiting) and will prevent situations described in this bug report.
