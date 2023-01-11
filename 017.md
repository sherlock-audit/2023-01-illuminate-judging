IllIllI

medium

# Swivel premium has extra slippage applied to it if not swapped

## Summary
Swivel premiums have extra slippage applied to them if they're not swapped


## Vulnerability Detail
The fix for [this](https://github.com/sherlock-audit/2022-10-illuminate-judging/issues/211) issue from the prior contest was to collect the premium, and provide the equivalent number of iPTs. The idea behind iPTs is that if you buy them on the open market, you'll get a discount, because the underlying is unavailable until the PT matures. In this case, no discount is given, and essentially the user just has their funds locked for no yield, until the PT matures.


## Impact
The Swivel premium is used to over-pay for iPTs, which is a form of slippage, and thus the user will have lost principal


## Code Snippet
If the premium is not swapped, it's used to mint iPTs directly, without any discount:
```solidity
// File: src/Lender.sol : Lender.lend()   #1

532                if (e) {
533                    // Swap the premium for Illuminate principal tokens
534                    swapped += yield(
535                        u,
536                        y,
537                        premium,
538                        msg.sender,
539                        IMarketPlace(marketPlace).markets(u, m, 0),
540                        premiumSlippage
541                    );
542                } else {
543                    // Send the premium to the redeemer to hold until redemption
544                    premiums[u][m] = premiums[u][m] + premium;
545    
546                    // Account for the premium
547 @>                 received = received + premium;
548                }
549    
550                // Mint Illuminate principal tokens to the user
551:@>             IERC5095(principalToken(u, m)).authMint(msg.sender, received);
```
https://github.com/sherlock-audit/2023-01-illuminate/blob/main/src/Lender.sol#L522-L558


## Tool used

Manual Review


## Recommendation
The premium should be sent back to the caller, rather than being used to create non-discounted iPTs. While this will use slightly more gas, the user has the option to do a swap instead, so it doesn't make sense to _not_ return the funds to them if they don't want iPTs for the premium

