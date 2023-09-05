Blunt Butter Moose

high

# Create2 failure is not checked
## Summary

The codebase does not check if the create2 has failed, leading to a loss of assets.

## Vulnerability Detail

If the create2 fails, it will return a zero address. Based on the error message, it is understood that the intention is to check if the returned `contractAddress` is zero. However, the check was mistakenly performed against the `symmioAddress` instead. 

Thus, the function will not revert even if the deployment via create2 fails.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/multiAccount/MultiAccount.sol#L110

```solidity
File: MultiAccount.sol
098:     function _deployContract(
099:         bytes memory bytecode,
100:         bytes32 salt
101:     ) internal returns (address contractAddress) {
102:         assembly {
103:             contractAddress := create2(
104:                 0,
105:                 add(bytecode, 32),
106:                 mload(bytecode),
107:                 salt
108:             )
109:         }
110:         require(symmioAddress != address(0), "MultiAccount: create2 failed");
111:         emit DeployContract(msg.sender, contractAddress);
112:         return contractAddress;
113:     }
```

As a result, a new PartyA with zero address will be created and added to the caller account.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/multiAccount/MultiAccount.sol#L125

```solidity
File: MultiAccount.sol
125:     function addAccount(string memory name) external whenNotPaused {
126:         address account = _deployPartyA();
127:         indexOfAccount[account] = accounts[msg.sender].length;
128:         accounts[msg.sender].push(Account(account, name));
129:         owners[account] = msg.sender;
130:         emit AddAccount(msg.sender, account, name);
131:     }
```

Assume the caller (Bob) is simply a normal trader and not an EVM/solidity programmer. It is important to understand that a normal person would not know that zero is a burn address in Ethereum. Thus, Bob assumes that account zero is the first account in the system and does not see any problem. 

He deposits assets into his account zero via the `MultiAccount.depositForAccount` function, which will work normally with a revert. Since the `AccountFacetImpl.deposit` function does not perform any zero-account check, the deposit will be successful, and the account zero's balance will be incremented by the deposited amount at Line 24 below.

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/facets/Account/AccountFacetImpl.sol#L19

```solidity
File: AccountFacetImpl.sol
19:     function deposit(address user, uint256 amount) internal {
20:         GlobalAppStorage.Layout storage appLayout = GlobalAppStorage.layout();
21:         IERC20(appLayout.collateral).safeTransferFrom(msg.sender, address(this), amount);
22:         uint256 amountWith18Decimals = (amount * 1e18) /
23:         (10 ** IERC20Metadata(appLayout.collateral).decimals());
24:         AccountStorage.layout().balances[user] += amountWith18Decimals;
25:     }
```

At this point, Bob's funds will be stuck due to the following reasons:

- Unable to withdraw the assets from his account zero because the withdrawal `MultiAccount.withdrawFromAccount` function will make an inner call to zero address, which will result in a revert
- Unable to perform any trade because `MultiAccount._call` function will make an inner call to zero address, which will result in a revert

## Impact

Assets will be stuck, as mentioned above.

## Code Snippet

https://github.com/sherlock-audit/2023-08-symmetrical/blob/main/symmio-core/contracts/multiAccount/MultiAccount.sol#L110

## Tool used

Manual Review

## Recommendation

Consider making the following change:

```diff
function _deployContract(
    bytes memory bytecode,
    bytes32 salt
) internal returns (address contractAddress) {
    assembly {
        contractAddress := create2(
            0,
            add(bytecode, 32),
            mload(bytecode),
            salt
        )
    }
+    require(contractAddress != address(0), "MultiAccount: create2 failed");
-    require(symmioAddress != address(0), "MultiAccount: create2 failed");
    emit DeployContract(msg.sender, contractAddress);
    return contractAddress;
}
```