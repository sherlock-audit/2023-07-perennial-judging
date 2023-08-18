Fresh Aegean Sparrow

medium

# Exiting the market as a trader results in a deficit in collateral balance.
## Summary
It is expected that a trader that closes all his position and withdraws all his collateral will have a collateral balance of 0, but this is not so due to the delayed settlement of position fees

## Vulnerability Detail
When a user updates his positions, the position fees and keeper fees does not get deducted immediately, rather it gets deducted during the next settlement.
If a trader decides to exit a market by closing all his positions and withdrawing all his collateral, the fees will be deducted during the next settlement, so his collateral balance becomes negative.
If trader does not enter the market again(very possible if trader is a contract controlled by an EOA), collateral balance of that account remains negative, so the assumption that account MUST pay back during the next settlement holds false.
When the fees get withdrawn by protocol, other users' collateral pay for the fees that the malicious trader ought to have paid.

Similarly, during liquidations, fees are deducted from the liquidated account's collateral balance during the next settlement, instead of it to be deducted from the liquidationFee that liquidator will receive. This is unfair to the liquidated account because now, they are paying liquidationFee+keeperFee+positionFee, instead of them to pay just liquidationFee.

## Impact
- Traders that permanently exit Market will not pay position fees, so other users are forced to pay it.
- Accounts being liquidated will pay liquidationFee+keeperFee+positionFee instead of just liquidationFee

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L345
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L440
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/types/Local.sol#L72

## Tool used

Manual Review

## Recommendation
When updating position, users should only be able to withdraw up to collateralBalance-(newOrder.fee+newOrder.keeper), so that when fees get debited during next settlement, collateralBalance will be 0 and not negative.
Also, during liquidation, liquidator should receive liquidatioFee-(newOrder.fee+newOrder.keeper)
