IllIllI

medium

# Ether price may be stale

## Summary
Ether price may be stale, limiting the usefulness of the rate limit feature


## Vulnerability Detail
if Ether price suddenly jumps and the admin is unable to update the price, or forgets to, and an attacker who was waiting for the price to go up before attacking attacks, the rate limit of 2m USD will not see the change in price, and the attacker will have been able to bypass it. If an attacker can just wait for a bull market before attacking, that defeats the purpose of having the check.

If the code had been using a Chainlink oracle, this issue would be equivalent to not checking whether the price was stale, which is a Medium-severity issue.

## Impact
An attacker will be able to make off with a lot more than they should be able to, meaning the rate limit feature can be bypassed under certain conditions


## Code Snippet
The rate limit depends on an admin setting the price of Ether, which may become stale:
```solidity
// File: src/Lender.sol : Lender.setEtherPrice()   #1

316        function setEtherPrice(uint256 p)
317            external
318            authorized(admin)
319            returns (bool)
320        {
321            etherPrice = p;
322            return true;
323:       }
```
https://github.com/sherlock-audit/2023-01-illuminate/blob/main/src/Lender.sol#L316-L323


## Tool used

Manual Review


## Recommendation
Use a chainlink oracle and check the last updated timestamp against a validity window

