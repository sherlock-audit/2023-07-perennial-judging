Fresh Aegean Sparrow

medium

# Several inconsistencies with payoffs especially if we are expecting base to be 1e6
## Summary
All the payoff contracts are not scaled similarly

## Vulnerability Detail
For Giga, payoff is calculated as price*1e15✅
For Kilo, payoff is calculated as price*1e9✅
For Mega, payoff is calculated as price\*1e12✅
For Micro, payoff is calculated as price/1e12❌
For Milli, payoff is calculated as price/1e9❌
For Nano, payoff is calculated as price/1e15❌

Normally, Here are the units of measurement:
Giga=1e9
Kilo=1e3
Mega=1e6
Micro=1e-6
Milli=1e-3
Nano=1e1e-9

From this, we can see that there are inconsistencies in scaling the price.

From the first three payoffs, payoff=price \* 1e6 \* unit of measurement
But in the last three, payoff=(price /1e6)/unit of measurement.

## Impact
Wrong payoff functions will lead to unintended and wrong scaling of oracle prices

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-payoff/contracts/payoff/Micro.sol#L10
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-payoff/contracts/payoff/Milli.sol#L10
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-payoff/contracts/payoff/Nano.sol#L10

## Tool used

Manual Review

## Recommendation
Stick to one scaling formula.
Use either of these for all the payoff functions

- payoff=price \* 1e6 \* unit of measurement or
- (price /1e6)/unit of measurement

Assuming payoff=price *1e6 * unit of measurement is the intended scaling formula,

Micro=price\*1e6\*1e-6=price(as against price/1e12)
Milli=price\*1e6\*1e-3=price*1e3(as against price/1e9)
Nano=price\*1e6\*1e-9=price/1e3(as against price/1e15)
