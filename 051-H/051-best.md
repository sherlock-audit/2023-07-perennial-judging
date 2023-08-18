Fresh Aegean Sparrow

high

# Attacker can drain Market.sol through liquidation
## Summary
`Market#_invariant` does not check if withdrawing a liquidation fee will not cause collateral balance of that account to go below 0.
This allows an attacker to open a position, and liquidate it multiple times before next settlement.
Even if deducting liquidation fee will cause the collateral balance of the account being liquidated to go below 0, liquidator is not stopped, so he can repeat this to drain Market.sol contract.
Attacker can setup a contract that repeats this liquidation process in a for loop, so as to perform the attack in a single transaction.

## Vulnerability Detail
Here is the relevant part of Market#\_invariant function:

```solidity
    function _invariant(
        Context memory context,
        address account,
        Order memory newOrder,
        Fixed6 collateral,
        bool protected
    ) private view {
        (Position[] memory pendingLocalPositions, Fixed6 collateralAfterFees) = _loadPendingPositions(context, account);

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

Protocol assumes that the check `collateral.lt(Fixed6Lib.from(-1, _liquidationFee(context)))` will prevent a liquidator from making an account's collateral less than 0, but if attacker repeats this before the next settlement, the check becomes ineffective as `_liquidationFee` is the same(because settlement has not occurred, so latestPosition has not changed):

```solidity
    function _liquidationFee(Context memory context) private view returns (UFixed6) {
        return context.latestPosition.local
            .liquidationFee(context.latestVersion, context.riskParameter)
            .min(UFixed6Lib.from(token.balanceOf()));
    }
```

This allows attacker to continue withdrawing liquidation fee from his collateral until he drains the Market.sol contract.

## Impact
Attacker can open a position that will easily get liquidatable, and when the position gets liquidatable, he would liquidate it many times until he drains Market.sol contract.
This is possible because the call won't revert when liquidationFee to be withdrawn is greater than the collateral balance of that account

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L465-L483

## Tool used

Manual Review

## Recommendation
Don't allow liquidations that will cause collateral balance of a position to go below 0.
Consider adding a ckeck like this:

```solidity
    function _invariant(
        Context memory context,
        address account,
        Order memory newOrder,
        Fixed6 collateral,
        bool protected
    ) private view {
        (Position[] memory pendingLocalPositions, Fixed6 collateralAfterFees) = _loadPendingPositions(context, account);

        if (protected && (
@>          (collateral.lt(Fixed6Lib.ZERO) && collateralAfterFees.lt(Fixed6Lib.ZERO))||
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
