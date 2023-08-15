Wonderful Silver Loris

high

# Invalid oracle versions can cause desync of global and local positions making protocol lose funds and being unable to pay back all users
## Summary

When oracle version is skipped for any reason (marked as invalid), pending positions are invalidated (reset to previous latest position):
```solidity
    function _processPositionGlobal(Context memory context, uint256 newPositionId, Position memory newPosition) private {
        Version memory version = _versions[context.latestPosition.global.timestamp].read();
        OracleVersion memory oracleVersion = _oracleVersionAtPosition(context, newPosition);
>        if (!oracleVersion.valid) newPosition.invalidate(context.latestPosition.global);
...
    function _processPositionLocal(
        Context memory context,
        address account,
        uint256 newPositionId,
        Position memory newPosition
    ) private {
        Version memory version = _versions[newPosition.timestamp].read();
>        if (!version.valid) newPosition.invalidate(context.latestPosition.local);
```

This invalidation is only temporary, until the next valid oracle version. The problem is that global and local positions can be settled with different next valid oracle version, leading to temporary desync of global and local positions, which in turn leads to incorrect accumulation of protocol values, mostly in profit and loss accumulation, breaking internal accounting: total collateral of all users can increase or decrease due to this while the funds deposited remain the same, possibly triggering a bank run, since the last user to withdraw will be unable to do so, or some users might get collateral reduced when it shouldn't (loss of funds for them).

## Vulnerability Detail

In more details, if there are 2 pending positions with timestamps different by 2 oracle versions and the first of them has invalid oracle version at its timestamp, then there are 2 different position flows possible depending on the time when the position is settled (update transaction called):
1. For earlier update the flow is: previous position (oracle v1) -> position 1 (oracle v2) -> position 2 (oracle v3)
2. For later update position 1 is skipped completely (the fees for the position are also not taken) and the flow is: previous position (oracle v1) -> invalidated position 1 (in the other words: previous position again) (oracle v2) -> position 2 (oracle v3)

While the end result (position 2) is the same, it's possible that pending global position is updated earlier (goes the 1st path), while the local position is updated later (goes the 2nd path). For a short time (between oracle versions 2 and 3), the global position will accumulate everything (including profit and loss) using the pending position 1 long/short/maker values, but local position will accumulate everything using the previous position with different values.

Consider the following scenario:
Oracle uses granularity = 100. Initially user B opens position maker = 2 with collateral = 100.
T=99:  User A opens long = 1 with collateral = 100 (pending position long=1 timestamp=100)
T=100: Oracle fails to commit this version, thus it becomes invalid
T=201: At this point oracle version at timestamp 200 is not yet commited, but the new positions are added with the next timestamp = 300:
User A closes his long position (update(0,0,0,0)) (pending position: long=1 timestamp=100; long=0 timestamp=300)
At this point, current global long position is still 0 (pending the same as user A local pending positions)

T=215: Oracle commits version with timestamp = 200, price = $100
T=220: User B settles (update(2,0,0,0) - keeping the same position).
At this point the latest oracle version is the one at timestamp = 200, so this update triggers update of global pending positions, and current latest global position is now long = 1.0 at timestamp = 200.
T=315: Oracle commits version with timestamp = 300, price = $90
after settlement of both UserA and UserB, we have the following:

1. Global position settlement. It accumulates position [maker = 2.0, long = 1.0] from timestamp = 200 (price=$100) to timestamp = 300 (price=$90). In particular:
longPnl = 1*($90-$100) = -$10
makerPnl = -longPnl = +$10
2. User B local position settlement. It accumulates position [maker = 2.0] from timestamp = 200 to timestamp = 300, adding makerPnl ($10) to user B collateral. So user B collateral = $110
3. User A local position settlement. When accumulating, pending position 1 (long = 1, timestamp = 100) is invalidated to previous position (long = 0) and also fees are set to 0 by invalidation. So user A local accumulates position [long = 0] from timestamp = 0 to timestamp = 300 (next pending position), this doesn't change collateral at all (remains $100). Then the next pending position [long = 0] becomes the latest position (basically position of long=1 was completely ignored as if it has not existed).

Result:
User A deposited $100, User B deposited $100 (total $200 deposited)
after the scenario above:
User A has collateral $110, User B has collateral $100 (total $210 collateral withdrawable)
However, protocol only has $200 deposited. This means that the last user will be unable to withdraw the last $10 since protocol doesn't have it, leading to a user loss of funds.

## Impact

Any time the oracle skips a version (invalid version), it's likely that global and local positions for different users who try to trade during this time will desync, leading to messed up accounting and loss of funds for users or protocol, potentially triggering a bank run with the last user being unable to withdraw all funds.

The severity of this issue is high, because while invalid versions are normally a rare event, however in the current state of the codebase there is a bug that pyth oracle requests are done using this block timestamp instead of granulated future time (as positions do), which leads to invalid oracle versions almost for all updates (that bug is reported separately). Due to this other bug, the situation described in this issue will arise very often by itself in a normal flow of the user requests, so it's almost 100% that internal accounting for any semi-active market will be broken and total user collateral will deviate away from real deposited funds, meaning the user funds loss.

But even with that other bug fixed, the invalid oracle version is a normal protocol event and even 1 such event might be enough to break internal market accounting.

## Proof of concept

The scenario above is demonstrated in the test, add this to test/unit/market/Market.test.ts:
```solidity
it('panprog global-local desync', async () => {
    const positionMaker = parse6decimal('2.000')
    const positionLong = parse6decimal('1.000')
    const collateral = parse6decimal('100')

    const oracleVersion = {
        price: parse6decimal('100'),
        timestamp: TIMESTAMP,
        valid: true,
    }
    oracle.at.whenCalledWith(oracleVersion.timestamp).returns(oracleVersion)
    oracle.status.returns([oracleVersion, oracleVersion.timestamp + 100])
    oracle.request.returns()

    dsu.transferFrom.whenCalledWith(userB.address, market.address, collateral.mul(1e12)).returns(true)
    await market.connect(userB).update(userB.address, positionMaker, 0, 0, collateral, false)

    const oracleVersion2 = {
        price: parse6decimal('100'),
        timestamp: TIMESTAMP + 100,
        valid: true,
    }
    oracle.at.whenCalledWith(oracleVersion2.timestamp).returns(oracleVersion2)
    oracle.status.returns([oracleVersion2, oracleVersion2.timestamp + 100])
    oracle.request.returns()

    dsu.transferFrom.whenCalledWith(user.address, market.address, collateral.mul(1e12)).returns(true)
    await market.connect(user).update(user.address, 0, positionLong, 0, collateral, false)

    var info = await market.locals(userB.address);
    console.log("collateral deposit maker: " + info.collateral);
    var info = await market.locals(user.address);
    console.log("collateral deposit long: " + info.collateral);

    // invalid oracle version
    const oracleVersion3 = {
        price: 0,
        timestamp: TIMESTAMP + 200,
        valid: false,
    }
    oracle.at.whenCalledWith(oracleVersion3.timestamp).returns(oracleVersion3)

    // next oracle version is valid
    const oracleVersion4 = {
        price: parse6decimal('100'),
        timestamp: TIMESTAMP + 300,
        valid: true,
    }
    oracle.at.whenCalledWith(oracleVersion4.timestamp).returns(oracleVersion4)

    // still returns oracleVersion2, because nothing commited for version 3, and version 4 time has passed but not yet commited
    oracle.status.returns([oracleVersion2, oracleVersion4.timestamp + 100])
    oracle.request.returns()

    // reset to 0
    await market.connect(user).update(user.address, 0, 0, 0, 0, false)

    // oracleVersion4 commited
    oracle.status.returns([oracleVersion4, oracleVersion4.timestamp + 100])
    oracle.request.returns()

    // settle
    await market.connect(userB).update(userB.address, positionMaker, 0, 0, 0, false)

    const oracleVersion5 = {
        price: parse6decimal('90'),
        timestamp: TIMESTAMP + 400,
        valid: true,
    }
    oracle.at.whenCalledWith(oracleVersion5.timestamp).returns(oracleVersion5)
    oracle.status.returns([oracleVersion5, oracleVersion5.timestamp + 100])
    oracle.request.returns()

    // settle
    await market.connect(userB).update(userB.address, positionMaker, 0, 0, 0, false)
    await market.connect(user).update(user.address, 0, 0, 0, 0, false)

    var info = await market.locals(userB.address);
    console.log("collateral maker: " + info.collateral);
    var info = await market.locals(user.address);
    console.log("collateral long: " + info.collateral);
})
```

Console output for the code:
```solidity
collateral deposit maker: 100000000
collateral deposit long: 100000000
collateral maker: 110000028
collateral long: 100000000
```
Maker has a bit more than $110 in the end, because he also earns funding and interest during the short time when ephemeral long position is active (but user A doesn't pay these fees).

## Code Snippet

`_processPositionGlobal` invalidates position if oracle version is invalid for its timestamp:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L390-L393

`_processPositionLocal` does the same:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L430-L437

`_settle` loops over global and local positions until the latest oracle version timestamp. In this loop each position is invalidated to previous latest if it has invalid oracle timestamp. So if `_settle` is called after the invalid timestamp, previous latest is accumulated for it:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L333-L347

Later in the `_settle`, the latest global and local position are advanced to latestVersion timestamp, the difference from the loop is that since position timestamp is set to valid oracle version, `_processPositionGlobal` and `_processPositionLocal` here will be called with valid oracle and thus position (which is otherwise invalidated in the loop) will be valid and set as the latest position:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L349-L360

This means that for early timestamps, invalid version positions will become valid in the `sync` part of the `_settle`. But for late timestamps, invalid version position will be skipped completely in the loop before `sync`. This is the core reason of desync between local and global positions.

## Tool used

Manual Review

## Recommendation

The issue is that positions with invalid oracle versions are ignored until the first valid oracle version, however the first valid version can be different for global and local positions. One of the solutions I see is to introduce a map of position timestamp -> oracle version to settle, which will be filled by global position processing. Local position processing will follow the same path as global using this map, which should eliminate possibility of different paths for global and local positions.

It might seem that the issue can only happen with exactly 1 oracle version between invalid and valid positions. However, it's also possible that some non-requested oracle versions are commited (at some random timestamps between normal oracle versions) and global position will go via the route like t100[pos0]->t125[pos1]->t144[pos1]->t200[pos2] while local one will go t100[pos0]->t200[pos2] OR it can also go straight to t300 instead of t200 etc. So the exact route can be anything, and local oracle will have to follow it, that's why I suggest a path map.

There might be some other solutions possible.