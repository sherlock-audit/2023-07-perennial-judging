Wonderful Silver Loris

high

# Bad debt (shortfall) liquidation leaves liquidated user in a negative collateral balance which can cause bank run and loss of funds for the last users to withdraw

Bad debt liquidation leaves liquidated user with a negative collateral. However, there is absolutely no incentive for anyone to repay this bad debt. This means that most of the time the account with negative balance will simply be abandoned. This means that this negative balance (bad debt) is taken from the other users, however it is not socialized, meaning that the first users to withdraw will be able to do so, but the last users will be unable to withdraw because protocol won't have enough funds. This means that any large bad debt in the market can trigger a bank run with the last users to withdraw losing their funds.

## Vulnerability Detail

Consider the following scenario:
1. User1 and User2 are the only makers in the market each with maker=50 position and each with collateral=500. (price=$100)
2. A new user comes into the market and opens long=10 position with collateral=10.
3. Price drops to $90. Some liquidator liquidates the user, taking $10 liquidation fee. User is now left with the negative collateral = -$100
4. Since User1 and User2 were the other party for the user, each of them has a profit of $50 (both users have collateral=550)
5. At this point protocol has total funds from deposit of User1($500) + User2($500) + new user($10) - liquidator($10) = $1000. However, User1 and User2 have total collateral of 1100.
6. User1 closes position and withdraws $550. This succeeds. Protocol now has only $450 funds remaining and 550 collateral owed to User2.
7. User2 closes position and tries to withdraw $550, but fails, because protocol doesn't have enough funds. User2 can only withdraw $450, effectively losing $100.

Since all users know about this feature, after bad debt they will race to be the first to withdraw, triggering a bank run.

## Impact

After **ANY** bad debt, the protocol collateral for all non-negative users will be higher than protocol funds available, which can cause a bank run and a loss of funds for the users who are the last to withdraw.

Even if someone covers the shortfall for the user with negative collateral, this doesn't guarantee absence of bank run:
1. If the shortfall is not covered quickly for any reason, the other users can notice disparency between collateral and funds in the protocol and start to withdraw
2. It is possible that bad debt is so high that any entity ("insurance fund") just won't have enough funds to cover it.

## Proof of concept

The scenario above is demonstrated in the test, add this to test/unit/market/Market.test.ts:
```solidity
it('panprog bad debt liquidation bankrun', async () => {

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
        maintenance: parse6decimal('0.01'),
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

    var collateral = parse6decimal('500')
    dsu.transferFrom.whenCalledWith(userB.address, market.address, collateral.mul(1e12)).returns(true)
    await market.connect(userB).update(userB.address, parse6decimal('50.000'), 0, 0, collateral, false)
    dsu.transferFrom.whenCalledWith(userC.address, market.address, collateral.mul(1e12)).returns(true)
    await market.connect(userC).update(userC.address, parse6decimal('50.000'), 0, 0, collateral, false)

    var collateral = parse6decimal('10')
    dsu.transferFrom.whenCalledWith(user.address, market.address, collateral.mul(1e12)).returns(true)
    await market.connect(user).update(user.address, 0, parse6decimal('10.000'), 0, collateral, false)

    var info = await market.locals(user.address);
    var infoB = await market.locals(userB.address);
    var infoC = await market.locals(userC.address);
    console.log("collateral before liquidation: " + info.collateral + " + " + infoB.collateral + " + " + infoC.collateral + " = " + 
        info.collateral.add(infoB.collateral).add(infoC.collateral));

    setupOracle('100', TIMESTAMP + 100, TIMESTAMP + 200);
    setupOracle('90', TIMESTAMP + 200, TIMESTAMP + 300);
    // liquidate
    const EXPECTED_LIQUIDATION_FEE = parse6decimal('10')
    dsu.transfer.whenCalledWith(liquidator.address, EXPECTED_LIQUIDATION_FEE.mul(1e12)).returns(true)
    dsu.balanceOf.whenCalledWith(market.address).returns(COLLATERAL.mul(1e12))
    await market.connect(liquidator).update(user.address, 0, 0, 0, EXPECTED_LIQUIDATION_FEE.mul(-1), true)

    setupOracle('90', TIMESTAMP + 200, TIMESTAMP + 300);
    await market.connect(userB).update(userB.address, 0, 0, 0, 0, false)
    await market.connect(userC).update(userC.address, 0, 0, 0, 0, false)

    var info = await market.locals(user.address);
    var infoB = await market.locals(userB.address);
    var infoC = await market.locals(userC.address);
    console.log("collateral after liquidation: " + info.collateral + " + " + infoB.collateral + " + " + infoC.collateral + " = " + 
        info.collateral.add(infoB.collateral).add(infoC.collateral));
})
```

Console output for the code:
```solidity
collateral before liquidation: 10000000 + 500000000 + 500000000 = 1010000000
collateral after liquidation: -100000080 + 550000000 + 550000000 = 999999920
```
After initial total deposit of $1010, in the end liquidated user will just abandon his account, and remaining user accounts have $550+$550=$1100 but only $1000 funds in the protocol to withdraw.

## Code Snippet

When account is liquidated, its collateral becomes negative, but is allowed when protected and the collateral is never reset to 0:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L497

However, user with negative collateral will simply abandond the account and shortfall will make it impossible to withdraw for the last users in case of bank run.

## Tool used

Manual Review

## Recommendation

There should be no negative collateral accounts with 0-position and no incentive to cover shortfall. When liquidated, if account is left with negative collateral, the bad debt should be added to the opposite position pnl (long position bad debt should be socialized between short position holders) or maybe to makers pnl only (socialized between makers). The account will have to be left with collateral = 0.

Implementation details for such solution can be tricky due to settlement in the future (pnl is not known at the time of liquidation initiation). Possibly a 2nd step of bad debt liquidation should be added: a keeper will call the user account to socialize bad debt and get some reward for this. Although this is not the best solution, because users who close their positions before the keeper socializes the bad debt, will be able to avoid this social loss. One of the solutions for this will be to introduce delayed withdrawals and delayed socialization (like withdrawals are allowed only after 5 oracle versions and socialization is applied to all positions opened before socialization and still active or closed within 5 last oracle versions), but it will make protocol much more complicated.