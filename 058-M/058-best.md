Fresh Aegean Sparrow

medium

# Malicious user can claim dust amount numerous times to cause Vault to lose through fees
Market.sol charges a fixed settlement fee each time a new order is created.
Vault calls Market#update each time a depositor claims assets(no matter how little).
A malicious user can claim dust amounts(as little as 0.00001 dollar) multiple times, and in a single transaction, to cause Vault to lose settlementFee\*numberOfTimesUserClaimed from each underlying market.

## Vulnerability Detail
Market.sol charges a fixed keeper fee each time a new order is created.
Order.sol:

```solidity
function registerFee()internal pure{
    ...
    self.keeper = isEmpty(self) ? UFixed6Lib.ZERO : marketParameter.settlementFee;
}
```

For a new order to be created, the new maker position should not be equal to the previous maker position.
Position.sol:

```solidity
    function update() internal pure returns (Order memory newOrder) {
        (newOrder.maker, newOrder.long, newOrder.short) = (
            Fixed6Lib.from(newMaker).sub(Fixed6Lib.from(self.maker)),
            Fixed6Lib.from(newLong).sub(Fixed6Lib.from(self.long)),
            Fixed6Lib.from(newShort).sub(Fixed6Lib.from(self.short))
        );
    }
```

StrategyLib#allocate determines the new maker positon that the vault will open whenever a user interacts with the vault.

```solidity
function allocate(){
    ...
    for (uint256 marketId; marketId < registrations.length; marketId++) {
        ...
        _locals.marketAssets = assets
            .muldiv(registrations[marketId].weight, totalWeight)
            .min(_locals.marketCollateral.mul(LEVERAGE_BUFFER));
        ...
        (targets[marketId].collateral, targets[marketId].position) = (
            Fixed6Lib.from(_locals.marketCollateral).sub(contexts[marketId].local.collateral),
            _locals.marketAssets
                .muldiv(registrations[marketId].leverage, contexts[marketId].latestPrice)
                .min(_locals.maxPosition)
                .max(_locals.minPosition)
        );
    }
}
```

From this, we can deduce that when |deltaChangeInAssets\*marketWeight/totalMarketWeight|>=1, a new order will be created in that underlying market, which will charge a fixed keeper fee.

Instead of claiming all his assets at once, which would charge keeper fee only once, The malicious user can claim deltaChangeInAssets x times to cause Vault to be charged keeperFee\*x.

## Impact
This would cause many users to be unable to withdraw their funds due to no more funds(as abnormally high keeper fees has been paid by vault), depending on the number of times the malicious user performs the attack

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L354-L371
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L87
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L273
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/types/Position.sol#L164

## Tool used

Manual Review

## Recommendation
I would suggest that when claiming assets, all the claimable assets(redeemed shares) for that account should be sent to him, instead of allowing the user to specify the amount he wants to claim.
In my opinion, this will do more good than harm.