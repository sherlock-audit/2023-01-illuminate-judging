IllIllI

high

# Users can make Illuminate pay their Swivel fees, locking Illuminate's other fees

## Summary
Swivel fees are taken out of Illuminate's fees which are stored in the `Lender` contract's balance, rather than from the user


## Vulnerability Detail
The Swivel version of `lend()` only transfers enough underlying from the user to cover the amounts lent, but does not calculate the fees associated with Swivel's lending. Swivel charges and transfers a separate fee on top of what's lent for some orders, and this fee will be transferred from the `Lender` contract, from whatever balance it holds. 


## Impact
Since the user will not have transferred the fee portion of the operation, the fee will be taken out of the remaining balance of the `Lender` contract, which will be the Illuminate fees held by the contract, meaning Illuminate will lose its fees. If the Swivel fees happen to be large, it's possible that _all_ Illuminate fees for that underlying will be taken. 

In addition, since the fees state variable will no longer match the contract balance, even if one extra wei of underlying is all that's stolen, fees won't be able to be withdrawn by the admin in `withdrawFee()`, since the full fee amount is always withdrawn, until more underlying is manually transferred to make the balance match.


## Code Snippet
The amount transferred comes from the `a` array, and is the amount lent, not including the fee portion: 
```solidity
// File: src/Lender.sol : Lender.lend()   #1

467        /// @param a array of amounts of underlying tokens lent to each order in the orders array
...
474        function lend(
...
478            uint256[] memory a,
479            address y,
480            Swivel.Order[] calldata o,
481            Swivel.Components[] calldata s,
482            bool e,
483            uint256 premiumSlippage
484        ) external nonReentrant unpaused(u, m, p) matured(m) returns (uint256) {
485            // Ensure all the orders are for the underlying asset
486            swivelVerify(o, u);
487    
488            // Lent represents the total amount of underlying to be lent
489 @>         uint256 lent = swivelAmount(a);
490    
...
494            // Transfer underlying token from user to Illuminate
495 @>         Safe.transferFrom(IERC20(u), msg.sender, address(this), lent);
496:   
```
https://github.com/sherlock-audit/2023-01-illuminate/blob/main/src/Lender.sol#L476-L496


The Illuminate fee is removed from the last amount in the array, but the array is otherwise unmodified. Note that the removal of the Illuminate fee (not the Swivel fee) is expected to be accounted for by the orders array being passed:
```solidity
// File: src/Lender.sol : Lender.lend()   #2

518                // Fill the given orders on Swivel
519:@>             ISwivel(swivelAddr).initiate(o, a, s);
```
https://github.com/sherlock-audit/2023-01-illuminate/blob/main/src/Lender.sol#L509-L529


Swivel iterates over the orders passed to it in `initiate()`, and passes along the unmodified amounts...:
```solidity
    function initiate(
        Hash.Order[] calldata o,
        uint256[] calldata a,
        Sig.Components[] calldata c
    ) external returns (bool) {
        // for each order filled, routes the order to the right interaction depending on its params
        for (uint256 i; i != o.length; ) {
            if (!o[i].exit) {
                if (!o[i].vault) {
                    initiateVaultFillingZcTokenInitiate(o[i], a[i], c[i]);
                } else {
                    initiateZcTokenFillingVaultInitiate(o[i], a[i], c[i]);
                }
            } else {
                if (!o[i].vault) {
                    initiateZcTokenFillingZcTokenExit(o[i], a[i], c[i]);
                } else {
                    initiateVaultFillingVaultExit(o[i], a[i], c[i]);
```
https://github.com/Swivel-Finance/swivel/blob/3cc31302f84c2b1777a53c11b22c58ec6ef17888/contracts/v3/src/Swivel.sol#L114-L131

...and some of the functions called either add the fee on top of the amount for the single transfer it does...:
```solidity
        Safe.transferFrom(uToken, msg.sender, address(this), (a + fee));
```
https://github.com/Swivel-Finance/swivel/blob/3cc31302f84c2b1777a53c11b22c58ec6ef17888/contracts/v3/src/Swivel.sol#L250

...or makes a separate transfer for the fee:
```solidity
        Safe.transferFrom(uToken, msg.sender, address(this), fee);
```
https://github.com/Swivel-Finance/swivel/blob/3cc31302f84c2b1777a53c11b22c58ec6ef17888/contracts/v3/src/Swivel.sol#L316


## Tool used

Manual Review


## Recommendation
Calculate the Swivel fee and transfer that too, or disallow Swivel orders where there is an extra fee assessed.

