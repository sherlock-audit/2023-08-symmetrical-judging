# Issue H-1: `liquidatePartyA` requires signature which doesn't have nonce, making possible unfair liquidation and loss of funds for all parties 

Source: https://github.com/sherlock-audit/2023-08-symmetrical-judging/issues/5 

## Found by 
panprog, xiaoming90

`liquidationSig` provided by liquidator to `liquidatePartyA` doesn't have nonces of neither partyA nor partyB. This means that this signature is valid even if partyA and/or partyB do some actions before the liquidation with this signature happens. There are numerous scenarios possible, both happening by itself and caused by malicious parties, where this can lead to loss of funds for some or all parties involved.

## Vulnerability Detail

Example Scenario:

1. PartyA: allocated=109, upnl=-99, cva+lf=10. To avoid liquidation, partyA tries to close 50% of its position, submitting requestToClosePosition
2. PartyA: allocated=109, upnl=-100, cva+lf=10. PartyA becomes liquidatable
3. Liquidator submits liquidating transaction (with upnl=-100), but partyB at the same time fulfills partyA request
3. partyB transaction executes first, partyA has: allocated = 59, upnl=-50, cva+lf=5 (not liquidatable anymore)
4. Liquidator transaction executes (with upnl=-100):
4.1. Available Balance = 59-5 - 100 = -46
4.2. Liquidation type = OVERDUE (46 > 5), deficit = 41
4.3. Corresponding partyB receives 50 - 50*46/100 = 27
4.4. PartyA allocated balance is set to 0
4.5. Liquidators receives 0 (because it's OVERDUE liquidation)

The result: 
1. PartyA was solvent just before the liquidation, but was still liquidated, losing all funds
2. PartyB has received 27 instead of full 50 profit (even though partyA had enough funds)
3. Liquidator didn't receive any fee, although it should have received it as partyA had enough funds.

All 3 parties have lost funds. Protocol basically stole those funds.

## Impact

Unfair liquidation for partyA, loss of funds for liquidator and partyB in many possible situations during partyA liquidation - happening by itself or caused by malicious actors.

## Proof of Concept

Add this to any test, for example to `ClosePosition.behavior.ts`.

```ts
it("PartyA wrong upnl liquidation", async function () {
  const context: RunContext = this.context;

  this.user_allocated = decimal(119);
  this.hedger_allocated = decimal(1000);
  
  this.user = new User(this.context, this.context.signers.user);
  await this.user.setup();
  await this.user.setBalances(this.user_allocated, this.user_allocated, this.user_allocated);

  this.hedger = new Hedger(this.context, this.context.signers.hedger);
  await this.hedger.setup();
  await this.hedger.setBalances(this.hedger_allocated, this.hedger_allocated);

  this.liquidator = new User(this.context, this.context.signers.liquidator);
  await this.liquidator.setup();

  // open position (100 @ 10)
  await this.user.sendQuote(limitQuoteRequestBuilder()
    .positionType(PositionType.LONG)
    .quantity(decimal(100))
    .price(decimal(10))
    .cva(decimal(6)).lf(decimal(4)).mm(decimal(10))
    .build()
  );
  await this.hedger.lockQuote(1, 0, decimal(1));
  await this.hedger.openPosition(1, limitOpenRequestBuilder().filledAmount(decimal(100)).openPrice(decimal(10)).price(decimal(10)).build());

  var info = await this.user.getBalanceInfo();
  console.log("partyA allocated: " + info.allocatedBalances / 1e18 + " locked: " + info.totalLocked/1e18 + " pendingLocked: " + info.totalPendingLocked / 1e18);
  var info = await this.hedger.getBalanceInfo(this.user.getAddress());
  console.log("partyB allocated: " + info.allocatedBalances / 1e18 + " locked: " + info.totalLocked/1e18 + " pendingLocked: " + info.totalPendingLocked / 1e18);

  // price goes to 9, so user is in a loss of -100 (less fee), locked balance = 10, so liquidatable
  // user tries to close half of its position to save liquidation
  //await this.user.setBalances(this.user_allocated, this.user_allocated, this.user_allocated);
  await this.user.requestToClosePosition(
    1,
    limitCloseRequestBuilder().quantityToClose(decimal(50)).closePrice(decimal(9)).build(),
  );

  // liquidators obtains signature to liquidate
  var sig = await getDummyLiquidationSig("0x10", decimal(-100), [1], [decimal(9)], decimal(-100));
  // fix timestamp
  sig.timestamp = BigNumber.from(await sig.timestamp).add(5);

  // but before liquidation, partyB fullfills partyA close request
  await this.hedger.fillCloseRequest(
    1,
    limitFillCloseRequestBuilder()
      .filledAmount(decimal(50))
      .closedPrice(decimal(9))
      .build(),
  );

  var info = await this.user.getBalanceInfo();
  console.log("after closing: partyA allocated: " + info.allocatedBalances / 1e18 + " locked: " + info.totalLocked/1e18 + " pendingLocked: " + info.totalPendingLocked / 1e18);
  var info = await this.hedger.getBalanceInfo(this.user.getAddress());
  console.log("after closing: partyB allocated: " + info.allocatedBalances / 1e18 + " locked: " + info.totalLocked/1e18 + " pendingLocked: " + info.totalPendingLocked / 1e18);

  // liquidator starts liquidating partyA
  await context.liquidationFacet.connect(this.liquidator.signer).liquidatePartyA(
    this.user.signer.address,
    sig
  );
  await context.liquidationFacet.connect(this.liquidator.signer).setSymbolsPrice(
    this.user.signer.address,
    sig
  );
  await context.liquidationFacet.connect(this.liquidator.signer).liquidatePositionsPartyA(
    this.user.signer.address,
    [1]
  );

  var info = await this.user.getBalanceInfo();
  console.log("after liquidation: partyA allocated: " + info.allocatedBalances / 1e18 + " locked: " + info.totalLocked/1e18 + " pendingLocked: " + info.totalPendingLocked / 1e18);
  var info = await this.hedger.getBalanceInfo(this.user.getAddress());
  console.log("after liquidation: partyB allocated: " + info.allocatedBalances / 1e18 + " locked: " + info.totalLocked/1e18 + " pendingLocked: " + info.totalPendingLocked / 1e18);

});
```

Console execution result:
```js
partyA allocated: 109 locked: 20 pendingLocked: 0
partyB allocated: 1000 locked: 20 pendingLocked: 0
after closing: partyA allocated: 59 locked: 10 pendingLocked: 0
after closing: partyB allocated: 1050 locked: 10 pendingLocked: 0
after liquidation: partyA allocated: 0 locked: 0 pendingLocked: 0
after liquidation: partyB allocated: 1079.5 locked: 0 pendingLocked: 0
liquidator balance: 0
```
As can be noticed, partyA + partyB have 1109 total after closing, but only 1079.5 after liquidation.

If the `fillCloseRequest` is commented out, then liquidation produces correct result:
```js
partyA allocated: 109 locked: 20 pendingLocked: 0
partyB allocated: 1000 locked: 20 pendingLocked: 0
after liquidation: partyA allocated: 0 locked: 0 pendingLocked: 0
after liquidation: partyB allocated: 1106 locked: 0 pendingLocked: 0
liquidator balance: 3
```

## Code Snippet

Notice that there are no nonces of either party in the liquidation signature:
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibMuon.sol#L54-L67

## Tool used

Manual Review

## Recommendation

Include partyA and partyB nonces in the liquidation signature.

# Issue H-2: `liquidatePositionsPartyA` limits partyB loss to partyB allocated balance, which can lead to inflated partyB balance and loss of funds for protocol users 

Source: https://github.com/sherlock-audit/2023-08-symmetrical-judging/issues/6 

## Found by 
bitsurfer, panprog

If partyB has positions both in a high profit and in a high loss, so that total upnl is much smaller than individual positions upnl, then partyB can deallocate most (or even all) of its funds. During partyA liquidation, `liquidatePositionsPartyA` limits partyB loss to the **current** allocated balance of partyB. In this case there is a big difference of the final results depending on order of position liquidations:
1. If the 1st position is in a profit, then the profit is first applied to balanceB, and then position in loss correctly subtracts from it (there is enough partyB balance to fully cover position in loss).
2. If the 1st position is in a loss, then balance (which is smaller than position loss) is simply reduced to 0, but then the 2nd position (which is in a profit) is added in full, leading to inflated final balance of partyB.

If such liquidation happens (intentional or not), partyB will have much more funds than it should at the expense of the other users: protocol will owe users more funds than it has, which can lead to bank run and the last users being unable to withdraw.

## Vulnerability Detail

Example Scenario:

Position 1: upnl = -960, cva+lf = 10, mm=10
Position 2: upnl = +1000, cva+lf = 10, mm=10
Total cva+lf = 20, total mm=20
PartyB has allocated balance = 0 (available = 0 + 1000 - 960 = 40 >= 20 - not liquidatable)
Party A is liquidated:
1. Position 1 is liquidated.
`amount` = 960
`amountToDeduct` = 0 (because partyB allocated balance = 0)
partyB allocatedBalance remains 0
2. Position 2 is liquidated.
`amount` = 1000
partyB allocatedBalance increases by 1000 (is set to 1000)

The result: PartyB should have a balance of less than 60 (upnl+cva), but it has 1000 instead!

## Impact

In some situations during partyA liquidations, PartyB balance is increased significantly while no funds enter the protocol, meaning protocol debt to users increases and is greater than protocol funds. This can cause a bank run, since the last users to withdraw will be unable to do so.

## Code Snippet

`amountToDeduct` is calculated as minimum between partyB loss and partyB balance. This amount is deducted from **current** partyB allocated balance.
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L163-L166

## Tool used

Manual Review

## Recommendation

Calculate total (signed) pnl for positions for each partyB before applying it: `amountToDeduct` should be for all positions combined for a particular partyB instead of separate positions.

# Issue M-1: Wrong calculation of solvency in `fillCloseRequest` prevents the position from being closed even if the user is solvent after position closure 

Source: https://github.com/sherlock-audit/2023-08-symmetrical-judging/issues/9 

## Found by 
panprog, xiaoming90

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

# Issue M-2: Position value can fall below minimum acceptable quote value when partially closing positions requested to be closed in full 

Source: https://github.com/sherlock-audit/2023-08-symmetrical-judging/issues/12 

## Found by 
panprog

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

# Issue M-3: MultiAccount `depositAndAllocateForAccount` function doesn't scale the allocated amount correctly, failing to allocate enough funds 

Source: https://github.com/sherlock-audit/2023-08-symmetrical-judging/issues/15 

## Found by 
Hama, panprog, tvdung94, xiaoming90

This is an issue very similar to [issue 222](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/222) from previous audit contest, but in a `MultiAccount` smart contract.

`MultiAccount.depositAndAllocateForAccount` uses the same `amount` value both for `depositFor` and for `allocate`. However, deposit amount decimals are from the collateral token while allocate amount decimals = 18. This means that for USDC (decimals = 6), `depositAndAllocateForAccount` will deposit correct amount, but allocate amount which is 1e12 times smaller (dust amount).

## Vulnerability Detail

Internal accounting (allocatedBalances) are tracked as fixed numbers with 18 decimals, while collateral tokens can have different amount of decimals. This is correctly accounted for in `AccountFacet.depositAndAllocate`:
```solidity
    AccountFacetImpl.deposit(msg.sender, amount);
    uint256 amountWith18Decimals = (amount * 1e18) /
        (10 ** IERC20Metadata(GlobalAppStorage.layout().collateral).decimals());
    AccountFacetImpl.allocate(amountWith18Decimals);
```

But it is treated incorrectly in `MultiAccount.depositAndAllocateForAccount`:
```solidity
    ISymmio(symmioAddress).depositFor(account, amount);
    bytes memory _callData = abi.encodeWithSignature(
        "allocate(uint256)",
        amount
    );
    innerCall(account, _callData);
```

This leads to incorrect allocated amounts.

## Impact

Similar to 222 from previous audit contest, the user expects to have full amount deposited and allocated, but ends up with only dust amount allocated, which can lead to unexpected liquidations (for example, user is at the edge of liquidation, calls depositAndAllocate to improve account health, but is liquidated instead). For consistency reasons, since this is almost identical to 222, it should also be high.

## Code Snippet

The same amount is used for `depositFor` and `allocate`:
https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/multiAccount/MultiAccount.sol#L167-L173

## Tool used

Manual Review

## Recommendation

Scale amount correctly before allocating it:
```solidity
    ISymmio(symmioAddress).depositFor(account, amount);
+   uint256 amountWith18Decimals = (amount * 1e18) /
+       (10 ** IERC20Metadata(collateral).decimals());
    bytes memory _callData = abi.encodeWithSignature(
        "allocate(uint256)",
-       amount
+       amountWith18Decimals
    );
    innerCall(account, _callData);
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because internally the depositAndAllocateForAccount calls the allocate function which does not use scaled amount but the actual input amount



**MoonKnightDev**

No funds will be lost. The user simply needs to reallocate their balance. Therefore, the severity is not high since there's no loss of funds.

**nevillehuang**

Relating to this [comment](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/222#issuecomment-1653365144) in the previous contest, it has the same root cause and potential same consequence, so could be valid H.

**MoonKnightDev**

> Relating to this [comment](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/222#issuecomment-1653365144) in the previous contest, it has the same root cause and potential same consequence, so could be valid H.

The root cause differs. Here, you can easily rectify the allocatedBalance by allocating again. Moreover, no funds are lost

**nevillehuang**

> > Relating to this [comment](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/222#issuecomment-1653365144) in the previous contest, it has the same root cause and potential same consequence, so could be valid H.
> 
> The root cause differs. Here, you can easily rectify the allocatedBalance by allocating again. Moreover, no funds are lost

Ok can be valid M according to Impact mentioned in the submission.

