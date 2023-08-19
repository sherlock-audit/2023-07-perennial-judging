Fresh Aegean Sparrow

medium

# Positions will be unfairly liquidated immediately after an unpause
While the Market is paused, many positions can become undercollateralized as price of Market token changes.
When the contract gets unpaused, Users are not given the chance to make their positions healthy, which would lead to unfair liquidations.

## Vulnerability Detail
The update function has a `whenNotPaused modifier`:

```solidity
  function update(
        address account,
        UFixed6 newMaker,
        UFixed6 newLong,
        UFixed6 newShort,
        Fixed6 collateral,
        bool protect
    ) external nonReentrant whenNotPaused {
        ...
    }
```

There is no grace period immediately after an unpause when liquidations will be paused.
Due to this, immediately Market gets unpaused, liquidators will liquidate as many undercollateralized positions as possible, while position owners won't have the chance to make their positions healthy.

## Impact
There will be unfair liquidations when Market gets unpaused

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L83

## Tool used

Manual Review

## Recommendation
There should be a grace period immediately after an unpause during which liquidations will be paused.
This will allow position owners to make their positions healthy.