Fresh Aegean Sparrow

medium

# Users can open positions such that the liquidationFee is more than the collateral balance
## Summary
There is no lower limit to the amount of collateral that a user can deposit, so a user can use very tiny collateral to open a very risky highly leveraged position, and when it gets liquidated, liquidator will get paid `riskParameter.minLiquidationFee` which is more than the user's collateral balance

## Vulnerability Detail
In Market.sol, there is no minCollateral that a user is allowed to deposit.
LiquidationFee is calculated as follows:

```solidity
    function liquidationFee(
        Position memory self,
        OracleVersion memory latestVersion,
        RiskParameter memory riskParameter
    ) internal pure returns (UFixed6) {
        return
            maintenance(self, latestVersion, riskParameter)
                .mul(riskParameter.liquidationFee)
                .min(riskParameter.maxLiquidationFee)
                .max(riskParameter.minLiquidationFee);
    }
```

A user can deliberately deposit very tiny collateral(specifically such that `collateral-fees<minLiquidationFee`), open a risky position, and since liquidator is guaranteed minLiquidationFee, user will liquidate his account whenever it gets undercollateralized to receive minLiquidationFee(which is more than the collateral balance).
User can perform this many times, or open positions with many contracts at once, so that when they get liquidatable, he will be able to liquidate all of them in a single transaction, effectively stealing minLiquidationFee-collateralHeDeposited each time.

## Impact
Malicious user can open positions with `collateral balance<minLiquidationFee`, and when it gets liquidated, Market loses minLiquidationFee-collateralBalance, while the liquidator(in case it is the user) gains minLiquidationFee-collateralBalance.

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L482

## Tool used

Manual Review

## Recommendation
Within `_invariant` function, ensure that collateralAfterFees is >= minLiquidationFee.
