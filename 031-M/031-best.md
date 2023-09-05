Blunt Butter Moose

high

# PartyB can block liquidation by incrementing the nonce
## Summary

PartyB can block liquidation by incrementing the nonce.

## Vulnerability Detail

The liquidation of PartyB can still be blocked by incrementing the nonce as mentioned by the [original report](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/233) and its duplicates.

The `liquidatePartyB` function still relies on the `verifyPartyBUpnl` function, which uses the on-chain nonce of PartyB for signature verification.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L257

```solidity
File: LiquidationFacetImpl.sol
244:     function liquidatePartyB(
245:         address partyB,
246:         address partyA,
247:         SingleUpnlSig memory upnlSig
248:     ) internal {
249:         AccountStorage.Layout storage accountLayout = AccountStorage.layout();
250:         MAStorage.Layout storage maLayout = MAStorage.layout();
251:         QuoteStorage.Layout storage quoteLayout = QuoteStorage.layout();
252: 
253: @>      LibMuon.verifyPartyBUpnl(upnlSig, partyB, partyA);
```

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibMuon.sol#L154

```solidity
File: LibMuon.sol
137:     function verifyPartyBUpnl(
138:         SingleUpnlSig memory upnlSig,
139:         address partyB,
140:         address partyA
141:     ) internal view {
142:         MuonStorage.Layout storage muonLayout = MuonStorage.layout();
143: //        require(
144: //            block.timestamp <= upnlSig.timestamp + muonLayout.upnlValidTime,
145: //            "LibMuon: Expired signature"
146: //        );
147:         bytes32 hash = keccak256(
148:             abi.encodePacked(
149:                 muonLayout.muonAppId,
150:                 upnlSig.reqId,
151:                 address(this),
152:                 partyB,
153:                 partyA,
154: @>              AccountStorage.layout().partyBNonces[partyB][partyA],
155:                 upnlSig.upnl,
156:                 upnlSig.timestamp,
157:                 getChainId()
158:             )
159:         );
160:         verifyTSSAndGateway(hash, upnlSig.sigs, upnlSig.gatewaySignature);
161:     }

```

The following functions, but not limited to, allow PartyB to increment the nonce to block the liquidation:

- Old `deallocateForPartyB` function

- New `chargeFundingRate` function

## Impact

PartyB can block their accounts from being liquidated by liquidators. High impact as per the [original report](https://github.com/sherlock-audit/2023-06-symmetrical-judging/issues/233)

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L257

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/libraries/LibMuon.sol#L154

## Tool used

Manual Review

## Recommendation

In most protocols, whether an account is liquidatable is determined on-chain, and this issue will not surface. However, the architecture of Symmetrical protocol relies on off-chain and on-chain components to determine if an account is liquidatable, which can introduce a number of race conditions such as the one mentioned in this report.

Consider reviewing the impact of malicious users attempting to increment the nonce in order to block certain actions in the protocols since most functions rely on the fact that the on-chain nonce must be in sync with the signature's nonce and update the architecture/contracts of the protocol accordingly.