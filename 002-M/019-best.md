IllIllI

medium

# Users can accidentally lose funds during redemption

## Summary
Users can accidentally lose funds during redemption


## Vulnerability Detail
The `redeem()` function for iPTs does not ensure that each of the backing PTs have been redeemed for underlying yet.


## Impact
A user may redeem before all backing PTs are redeemed (e.g. due to them being paused, or the keeper not having completed its work), thinking everything is done since maturity has passed, and will not get the full amount of underlying they are owed.


## Code Snippet
The amount redeemed is a fraction of holdings, which may not have been increased to the full amount yet:
```solidity
// File: src/Redeemer.sol : Redeemer.redeem()   #1

508        function redeem(address u, uint256 m) external unpaused(u, m) {
509            // Get Illuminate's principal token for this market
510            IERC5095 token = IERC5095(
511                IMarketPlace(marketPlace).markets(
512                    u,
513                    m,
514                    uint8(MarketPlace.Principals.Illuminate)
515                )
516            );
517    
518            // Verify the token has matured
519            if (block.timestamp < token.maturity()) {
520                revert Exception(7, block.timestamp, m, address(0), address(0));
521            }
522    
523            // Get the amount of tokens to be redeemed from the sender
524            uint256 amount = token.balanceOf(msg.sender);
525    
526            // Calculate how many tokens the user should receive
527            uint256 redeemed = (amount * holdings[u][m]) / token.totalSupply();
528    
529            // Update holdings of underlying
530            holdings[u][m] = holdings[u][m] - redeemed;
531    
532            // Burn the user's principal tokens
533            token.authBurn(msg.sender, amount);
534    
535            // Transfer the original underlying token back to the user
536            Safe.transfer(IERC20(u), msg.sender, redeemed);
537    
538            emit Redeem(0, u, m, redeemed, amount, msg.sender);
539:       }
```
https://github.com/sherlock-audit/2023-01-illuminate/blob/main/src/Redeemer.sol#L508-L539

## Tool used

Manual Review


## Recommendation
This is hard to solve without missing corner cases, because each external PT may have its own idosyncratic reasons for delays, and there may be losses/slippage involved when redeeming for underlying. I believe the only way that wouldn't allow griefing, would be to track the number of external PTs of each type that were deposited for minting Illuminate PTs on a per-market basis, and require() that the number of each that have been redeemed equals the minting count, before allowing the redemption of any Illuminate PTs for that market. You would also need an administrator override that bypasses this check for specific external PTs of specific maturities.

This is the same issue as was found in the [prior](https://github.com/sherlock-audit/2022-10-illuminate-judging/issues/81) [contest](https://github.com/sherlock-audit/2022-10-illuminate-judging/issues/222), and has not been addressed. Note that in the escallation, Evert [stated](https://github.com/sherlock-audit/2022-10-illuminate-judging/issues/81#issuecomment-1327149249): `Escalation accepted, enforcing a specific order of contract calls off-chain is not secure. Rewarding a medium severity.`
