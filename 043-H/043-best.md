Wonderful Silver Loris

high

# PythOracle allows any user to commit non-requested oracle version for any timestamp after the previous commit even if it's long ago which can be abused to steal from the protocol via price manipulation and collateral removal

PythOracle allows any user to commit non-requested oracle version. The only restriction is for the commit timestamp specified by the user to be after the previous commit. If there were no commits nor oracle requests for some time, it is possible to manipulate the price to steal funds from protocol. Since collateral change is the only operation which can be done without first requesting a price, we can use it to "close" positions via removing max collateral at artificial price and abandoning it with a bad debt (additionally profiting from liquidating it).

## Vulnerability Detail

The theft scenario (consider leverage = 100 is allowed):
1. Use 2 accounts to open 1 ETH long and 1 ETH short positions at $1000, each with $100 collateral
2. Monitor pyth prices and record all prices with signatures every second, wait for the following conditions:
- current price is greater than $1020
- AND interval min price is less than $980 (this can also be reversed)
3. As soon as the conditions are met, commit oracle version at the timestamp of min price with that price (say, $980)
4. Remove $110 of collateral from short position (which has $20 profit from the latest oracle commit), leaving it with $10 collateral (at the $980 price user has commited). It passes, because user is now at leverage = 98, which is allowed.
5. Initiate closing long position and initiate liquidating short position (this creates oracle request)
Steps 3-5 can be done in a single transaction
6. Fulfill oracle request with a price = $1020.
7. Settle market: receive $20 profit from long position and remove $120 collateral from it, receive some liquidation fee from the short position, even though it's in a bad debt.

Result: from initial $200 deposit get back $230 + liquidation fee

There are a few conditions to be able to steal, but all of them are feasible, although not in all markets:
1. Market has a stale price check, so riskParameter.staleAfter should be sufficiently long to catch some volatility (depending on volatility and leverage, a few minutes to a few hours are required)
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L485-L486
2. The leverage should be sufficiently high to catch profitable price swings (leverage = 50-100 is quite enough for this and many markets can use such leverage)
3. The market should be relatively quiet (no trades and thus oracle requests and commits happening in the time interval)

## Impact

Each round of commit function abuse allows to steal a certain (high) percentage of the amount involved. With high enough amounts, this can lead to significant loss of funds (up to all funds market has) for all market participants.

## Code Snippet

PythOracle `commit()` function has only 2 checks. No oracle requests pending:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L167-L170
And commited oracle version to be later than previous commit:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L175

Additionally, when getting and validating pyth price from data, it should be between MIN and MAX valid time after the specified oracleVersion, but it's easy to overcome by choosing appropriate oracleVersion needed (slightly in the past from the price itself).
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L190-L195

Market **does NOT** request oracle version if there is no change to position (only collateral change):
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L284

## Tool used

Manual Review

## Recommendation

User should not be able to commit price for arbitrary timestamp in the past, he must be restricted to a more recent price only. For example, consider restricting `commit()`'s oracleVersion to be at max `GRACE_PERIOD` from current `block.timestamp`.