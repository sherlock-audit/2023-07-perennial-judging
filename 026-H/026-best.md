Fresh Aegean Sparrow

high

# User can get liquidated multiple times before next settlement
## Summary

Whenever an account's position becomes unhealthy, anyone is allowed to liquidate that account by withdrawing an amount of liquidationFee from the account's collateral and closing all open positions.
The problem is that, a liquidator can liquidate that account multiple times before the next settlement(i.e. before another oracleVersion is committed), withdrawing liquidationFee each time, to steal all that account's collateral.

## Vulnerability Detail

Within the Market#update function, especially `_invariant` function, there is no way to know if an account has just been liquidated, which allows reliquidation.

```solidity
function _invariant()private view{
    ...
    if (protected && (
            !context.currentPosition.local.magnitude().isZero() ||
            context.latestPosition.local.collateralized(
                context.latestVersion,
                context.riskParameter,
                collateralAfterFees.sub(collateral)
            ) ||
            collateral.lt(Fixed6Lib.from(-1, _liquidationFee(context)))
        )) revert MarketInvalidProtectionError();
    ...
}
```

When reliquidating an account before the next settlement, this check: `context.latestPosition.local.collateralized(context.latestVersion,context.riskParameter,collateralAfterFees.sub(collateral))` will pass because the latestPosition was not updated in the previous liquidation, so function still thinks that the account is still undercollateralized, and will allow liquidator to liquidate again.

## Impact

Liquidators will withdraw more collateral than they ought to from an account, effectively draining the account of all its collateral.
This is a high risk as ALL deposited collateral of a user will be stolen when it becomes liquidatable.
Also, liquidated account will pay higher keeperFees during next settlement. Instead of paying keeperFee, liquidated account pays keeperFee\*numberOfTimesLiquidatorLiquidated

## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L465-L483

## Tool used

Manual Review

## Recommendation

If the magnitude of the pendingPosition for that account is 0, revert as the position has already been liquidated.

```solidity
function _invariant()private view{
    (Position[] memory pendingLocalPositions, Fixed6 collateralAfterFees) = _loadPendingPositions(context, account);
@>  Position memory lastPendingLocalPosition=pendingLocalPositions[pendingLocalPositions.length-1]
    if (protected && (
@>          lastPendingLocalPosition.magnitude().isZero()||
            !context.currentPosition.local.magnitude().isZero() ||
            context.latestPosition.local.collateralized(
                context.latestVersion,
                context.riskParameter,
                collateralAfterFees.sub(collateral)
            ) ||
            collateral.lt(Fixed6Lib.from(-1, _liquidationFee(context)))
        )) revert MarketInvalidProtectionError();
    ...
}
```
