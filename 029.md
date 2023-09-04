Blunt Butter Moose

medium

# Signature hash collision
## Summary

The collision issues highlighted in [Issue 214](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/214) and [Issue 180](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/180) were not sufficiently remediated as the method name is not included in the data to be signed/hashed.

## Vulnerability Detail

The collision issues highlighted in [Issue 214](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/214) and [Issue 180](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/180) were found not sufficiently remediated.

The [Muon docs](https://dev.muon.net/#signparams) stressed the importance of including the method name in the data to be signed.

> To ensure that the signed and verified response has accurately covered the requested data, the parameters passed to the app should also be included in the returned value of signParams in addition to the result. Otherwise, the signature queried from the app with certain parameters might be abused and fed to the dApp contract with different ones. **If the app has different methods, the method name should be included as well**.

The fix of including the method name into the data to be signed/hashed is only applied to the `LibMuon.verifyLiquidationSig` function.

To completely remediate the issue and prevent any potential hash collision, the fix must be applied to all the functions.

## Impact

The current implementation is not adequately guarded against potential hash collision.

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibMuon.sol#L72

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibMuon.sol#L89

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibMuon.sol#L110

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibMuon.sol#L137

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibMuon.sol#L163

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibMuon.sol#L194

## Tool used

Manual Review

## Recommendation

Applied the fix to the following functions:

-  `LibMuon.verifyQuotePrices`
-  `LibMuon.verifyPartyAUpnl`
-  `LibMuon.verifyPartyAUpnlAndPrice`
-  `LibMuon.verifyPartyBUpnl`
-  `LibMuon.verifyPairUpnlAndPrice`
-  `LibMuon.verifyPairUpnl`

The method name must be consistently added to all the payloads, not a selected few.