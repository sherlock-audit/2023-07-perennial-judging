Agreeable Rainbow Crow

high

# Market: User can frontrun oracle version update to settle position at a determined price
## Summary

User can monitor price in oracle version update transaction. The price will be used in the next settlement. User can frontrun this transaction to settle his position at a determined price. So that entering or exiting positions can be timed to profit.

This volilates [the settlement design described in the doc](https://docs.perennial.finance/mechanism/settlement),

> Since the payoff is computed directly from the oracle price, lag needs to introduced into the settlement system so that entering and exiting positions cannot be timed to arbitrage the market.

## Vulnerability Detail

The doc described the settlement machenisms.

> Time in Perennial markets is split into units called oracle versions. Settlement only occurs on the transition of one oracle version to another. When a user requests to open or close a new position, that position delta sits in a pending state until the next oracle version update, after which it takes effect.

The user is able to commit the next oracle version update because committing is not restricted in PythOracle.sol. Or he can monitor the pool for the next oracle version update.

So the user knows the next settlement price.

The user can time his entering and exiting to profit by doing,

1. enter market
2. monitor oracle version update tx, so he knows next settlement price
3. if he will profit from the price, frontrun the oracle version update tx with exit requesting. This request will be pending
4. oracle version updates
5. his pending position change settles and he takes profit

## Impact

The settlement machenisms protecting arbitrage is broken. Attacker can arbitrage the market and profit.

## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L85

## Tool used

Manual Review

## Recommendation

Settle the pending positions behind one more oracle version.