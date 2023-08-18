Fresh Aegean Sparrow

medium

# Markets cannot use USDC as underlying token
## Summary
Due to the assumption that underlying collateral token will have 18 decimals, wrong scaling formula is used, which prevents the use of USDC that has 6 decimals

## Vulnerability Detail
To deposit collateral into the Market, an amount of collateral tokens is transferred from `msg.sender`. This amount is calculated using `UFixed18Lib.from(collateral.abs())`, which effectively multiplies the collateral value by 1e12.

For example, if the collateral is in DSU (with 18 decimal places), the smallest parameter value that can be passed is 1 (since Solidity doesn't support floats). In this case, the function transfers 1 \* 1e12 DSU tokens, which equals 1e-6 dollar.

However, if the collateral is in USDC (with 6 decimal places), the smallest parameter value that can be passed is also 1. In this situation, the function transfers 1 \* 1e12 USDC tokens, which is equivalent to 1e12 / 1e6 = 1e6 dollars.

Therefore, when using USDC as the underlying token, the minimum collateral amount that can be used is 1 million dollars, which is very expensive and unintended

## Impact
If underlying market token is USDC, the minimum amount that can be used as collateral is 1 million dollars(which is very costly). Therefore, it will be expensive for most people to use such market.

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L294-L295

## Tool used

Manual Review

## Recommendation

The collateral input parameter should be of type Fixed18, so that there will be no need for scaling.
Note that multiple parts of the contract that deals with sending, receiving or accounting of collateral must be modified as well e.g. claimFee function
