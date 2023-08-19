Brisk Orchid Elk

high

# StrategyLib.sol#L112 : `_loadContext` is not accumulating the market's `maintenance`

when loading the market context,  the parameters related to market context are updated. For each positions there will be a maintenance. But when loading the maintenance for each position, the values are not accumulated, instead, the last maintenance value is updated as maintenance from the function [_loadContext](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L100C14-L100C26)

The function `_loadContext` is called when calculating the allocation value for each market inside the function [allocate](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L59)

## Vulnerability Detail

Lets look at the function [_loadContext](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L100C14-L100C26),

```solidity
    function _loadContext(Registration memory registration) private view returns (MarketContext memory context) {
        context.marketParameter = registration.market.parameter();
        context.riskParameter = registration.market.riskParameter();
        context.local = registration.market.locals(address(this));
        context.currentAccountPosition = registration.market.pendingPositions(address(this), context.local.currentId);


        Global memory global = registration.market.global();


        context.latestPrice = global.latestPrice.abs();
        context.currentPosition = registration.market.pendingPosition(global.currentId);


        for (uint256 id = context.local.latestId; id < context.local.currentId; id++)
            context.maintenance = registration.market.pendingPositions(address(this), id) -------------->>> not accumulated here.
                .maintenance(OracleVersion(0, global.latestPrice, true), context.riskParameter)
                .max(context.maintenance);
    }
```

lets check where this `_loadContext` is called, in [allocate](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L59)
```solidity
    function allocate(
        Registration[] memory registrations,
        UFixed6 collateral,
        UFixed6 assets
    ) internal view returns (MarketTarget[] memory targets) {
        MarketContext[] memory contexts = new MarketContext[](registrations.length);
        for (uint256 marketId; marketId < registrations.length; marketId++)
            contexts[marketId] = _loadContext(registrations[marketId]); ------------------------>>> called here.


        (uint256 totalWeight, UFixed6 totalMaintenance) = _aggregate(registrations, contexts);
.
.
.
```

One of the place where the `allocate` function is used is, inside the vault contract when managing the  the internal collateral and position strategy of the vault. Refer the [line](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L359)

## Impact

Loss of maintanance. For single market it seems be small, but when it is accumulated over n number of markets, the loss would be big.

## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L59-L68

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L100-L115

## Tool used

Manual Review

## Recommendation

Update the function [_loadContext](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L100C14-L100C26) as shown below.
```solidity
    function _loadContext(Registration memory registration) private view returns (MarketContext memory context) {
        context.marketParameter = registration.market.parameter();
        context.riskParameter = registration.market.riskParameter();
        context.local = registration.market.locals(address(this));
        context.currentAccountPosition = registration.market.pendingPositions(address(this), context.local.currentId);


        Global memory global = registration.market.global();


        context.latestPrice = global.latestPrice.abs();
        context.currentPosition = registration.market.pendingPosition(global.currentId);


        for (uint256 id = context.local.latestId; id < context.local.currentId; id++)
            context.maintenance = registration.market.pendingPositions(address(this), id) ------>>> ------
            context.maintenance += registration.market.pendingPositions(address(this), id)----->>> ++++
                .maintenance(OracleVersion(0, global.latestPrice, true), context.riskParameter)
                .max(context.maintenance);
    }
```