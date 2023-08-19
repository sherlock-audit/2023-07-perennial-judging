Fresh Aegean Sparrow

medium

# Vault.sol: Underlying Market fee is not accounted for in Vault.sol
Whenever Vault deposits to an underlying market, market charges a fee based on the size of position that is opened.
However, in Vault.sol, depositors are not made to pay the fees. This allows depositors to be able to claim a slightly higher amount than they should, while a few other depositors(that withdraw their funds late) are forced to pay all the fees that Market has been charging the Vault with their collateral.

## Vulnerability Detail
Looking at `_settle` function,

```solidity
    function _settle(Context memory context) private {
        while (
            context.global.current > context.global.latest &&
            _mappings[context.global.latest + 1].read().ready(context.latestIds)
        ) {
            uint256 newLatestId = context.global.latest + 1;
            context.latestCheckpoint = _checkpoints[newLatestId].read();
@>          (Fixed6 collateralAtId, UFixed6 feeAtId, UFixed6 keeperAtId) = _collateralAtId(context, newLatestId);
            context.latestCheckpoint.complete(collateralAtId, feeAtId, keeperAtId);

            context.global.processGlobal(
                newLatestId,
                context.latestCheckpoint,
                context.latestCheckpoint.deposit,
                context.latestCheckpoint.redemption
            );
            _checkpoints[newLatestId].store(context.latestCheckpoint);
        }

        // settle local position
        if (
            context.local.current > context.local.latest &&
            _mappings[context.local.current].read().ready(context.latestIds)
        ) {
            uint256 newLatestId = context.local.current;
            Checkpoint memory checkpoint = _checkpoints[newLatestId].read();
            context.local.processLocal(
                newLatestId,
                checkpoint,
                context.local.deposit,
                context.local.redemption
            );
        }
    }
```

The fee that Market charged Vault is read via `_collateralAtId`

In Account#processGlobal, toAssetsGlobal and toSharesGlobal only account for the keeper fees, but does not account for position fees(maker ) fees.

When every depositor withdraws their assets, the last few will not be able to because Vault does not have enough funds as paid fees was not accounted for.

## Impact

Despite that an amount of fee has been deducted from a Vault depositor's assets, they can still claim their full deposits without being charged any fee.
This will affect some other vault depositors(specifically, the last few withdrawers) as they won't be able to withdraw part(or all, depending on the situation) of their collateral because it has been used to pay Market position fees

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L324
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L342

## Tool used

Manual Review

## Recommendation
Just like \_withoutKeeperGlobal, \_withoutKeeperLocal, and \_withoutKeeper functions, Checkpoint.sol should also have \_withoutPositionFeeGlobal, \_withoutPositionFeeLocal and \_withoutPositionFee, which would update funds that users deposited to account for the fees charged by underlying market