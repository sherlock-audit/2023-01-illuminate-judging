ck

high

# Redeemer::setConverter can result in multiple addresses being approved to transfer interest bearing tokens.

## Summary

The `Redeemer::setConverter` function when called more than once will lead to multiple addresses having approval to transfer interest bearing tokens. This is because the previous approvals are not revoked when a new converter is set.

## Vulnerability Detail

When a new converter is set using `Redeemer::setConverter`, approvals of the previous converters remain.

https://github.com/sherlock-audit/2023-01-illuminate/blob/main/src/Redeemer.sol#L155-L179

```solidity
 /// @notice sets the converter address
    /// @param c address of the new converter
    /// @param i a list of interest bearing tokens the redeemer will approve
    /// @return bool true if successful
    function setConverter(address c, address[] memory i)
        external
        authorized(admin)
        returns (bool)
    {
        // Set the new converter
        converter = c;

        // Have the redeemer approve the new converter
        for (uint256 x; x != i.length; ) {
            // Approve the new converter to transfer the relevant tokens
            Safe.approve(IERC20(i[x]), c, type(uint256).max);

            unchecked {
                x++;
            }
        }

        emit SetConverter(c);
        return true;
    }
```

This can cause loss of funds in various cases e.g when an incorrect converter address is specified or a previous converter is deemed malicious. 

## Impact

This can lead to loss of funds as all previous converters will still have maximum approval to transfer interest bearing tokens. Previous converters may either have been wrongfully set or no longer trusted.

## Code Snippet

https://github.com/sherlock-audit/2023-01-illuminate/blob/main/src/Redeemer.sol#L155-L179

## Tool used

Manual Review

## Recommendation

Provide functionality to remove the approval of previous converters.