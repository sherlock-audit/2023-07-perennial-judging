# Issue H-1: Oracle request timestamp and pending position timestamp mismatch can make most position updates invalid 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/42 

## Found by 
KingNFT, WATCHPUG, minhtrng, panprog

When a new pending position is added, its timestamp is set to `currentTimestamp` returned by oracle's `status` function, which is a timestamp at certain granularities rounding up into the future, which means that most of the time it's greater than `block.timestamp`. However, when `request` is called for the oracle, the request timestamp is set to `block.timestamp`. Due to this mismatch, when the oracle price is commited, it is commited with request's timestamp, but when the position is settled, it tries to read the price at position's timestamp, which is a different time. As such, if the oracle price is commited for each request, it's still easily possible that all pending positions will have invalid oracle versions, completely breaking the protocol's functionality.

## Vulnerability Detail

An example of what happens exactly:

1. PythOracle granularity is set to 100.
2. User opens position at timestamp = 101. Pending position is stored with timestamp = 200 (because PythOracle returns currentTimestamp = 200)
3. At the same time `oracle.request()` is called, which stores 101 (current timestamp) into `versionList`
4. User calls `oracle.commitRequested()`, which stores current price into `_prices[101]`
5. Later when that pending position is settled, it requests `oracle.at(200)` which doesn't have a price set (is invalid).

The same will happen to all pending positions - so most of them will easily be invalid, which will completely break the protocol and cause all kinds of problems due to pending positions being invalid and not updating profit and loss properly.

## Impact

The most straightforward impact is unexpectedly long position commit times and possible funds loss due to this, if the oracle commit flow is the normal expected flow (only commit requested versions). For example:
T=1:   User A requests to open position long = 1. Position timestamp = 100. Oracle request timestamp = 1
T=15:  Oracle commits requested version at timestamp = 1, price = $100.
T=10010: User B requests to open position. Position timestamp = 10100. Oracle request timestamp = 10010
T=10025: Oracle commits requested version at timestamp = 10010, price = $110.

User A expects to be filled at price close to $100. However, he's only filled when the next user trades after him, which happens much later than expected with a very different price ($110), so User A has lost $10 unexpectedly. Basically, each user will only be settled when the next user trades. In quiet markets this can lead to very long settlement times and very bad prices for users.

User A, however, can notice these long waiting times and can fix it by voluntary commiting non-requested versions. For example, he can commit at T=120 and be filled with the correct price. However, this will mean that all commits must be made non-requested, thus they will not be rewarded with the keeper fees. So the user will pay keeper fees when trading, but will also be forced to lose gas fees for oracle commits, so either broken and long waiting times, or broken oracle non-rewarded updates: both are high impacts.

Another impact is completely broken internal accounting due to a lot of invalid oracle versions. There is a different bug reported by me about desync of global and local positions during invalid oracles. This bug, when coupled with the desync of global and local positions, will lead to catastrophic consequences and complete breakage of accounting of collateral, bank run and loss of funds for users. Scenario of what can (and will) happen:
User B has active open position maker=2 with collateral = 100
T=99: User A opens long=1 with collateral=100: update(0,1,0,100) (pending position timestamp = 100)
T=101: User A decides to close: update(0,0,0,0) (pending position timestamp = 200)
T=130: Oracle commited for timestamp=110, price = $100 (user A position at timestamp = 100 is invalid)
T=150: User B settles: update(2,0,0,0)
T=220: Oracle commited for timestamp=205, price = $90
after settlement of user A and user B:
user A will have collateral = $100 (local pending position long = 1 at timestamp = 100 will be invalidated and ignored)
user B will have collateral = $110 (global pending position long = 1 will be current at timestamp 110 and accumulate pnl from timestamp 110 to timestamp=205)

So total deposit of both users is $100 + $100 = $200
Total collateral in the end: $100 + $110 = $210
But protocol only has $200 in funds, so users will be unable to withdraw everything, which can cause bank run and loss of funds for the last user.

Such situations will happen all the time by themselves due to lots of invalid oracle versions, so this will mess up accounting completely.

For the details of this bug, you can refer to my other report.

## Code Snippet

1. Oracle `status()` returns timestamp which is in the future.

Oracle `status()` returns timestamp directly from current provider's `status()`:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/Oracle.sol#L47

PythOracle `status()` timestamp is taken from `current()`, which in turn returns `current()` from PythFactory:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L101

PythFactory `current()` returns timestamp which is granulated into the future using ceilDiv, which rounds up:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythFactory.sol#L76

2. Pending position's timestamp is taken from oracle `status()`.

`context.currentTimestamp` is set to timestamp from `oracle.status()`:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L312
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L575

New pending positions (global and local) timestamp is set to `context.currentTimestamp`:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L267-L269

And `request()` from oracle is done at the same time:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L284

3. PythOracle `request()` stores `block.timestamp` in the request list (called `versionList`) (**not** `current()` timestamp):
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L77-L81

4. PythOracle `commitRequested()` sets price at `versionList` timestamp (i.e. `block.timestamp` at the time `request()` was made)

`versionToCommit` is stored request's timestamp:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L135

The commit price is stored at the `versionToCommit` timestamp:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L154
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L202-L203

## Tool used

Manual Review

## Recommendation

Make timestamp of pending positions and timestamp of oracle request match. Record `current()` as a timestamp for the `request()`:
```solidity
 function request(address) external onlyAuthorized { 
     uint nextTimestamp = current();
     if (versionList.length == 0 || versionList[versionList.length - 1] < nextTimestamp) { 
         versionList.push(nextTimestamp);
     } 
 } 
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> h



# Issue H-2: PythOracle:if price.expo is less than 0, wrong prices will be recorded 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/56 

## Found by 
Emmanuel, minhtrng, panprog
In PythOracle#\_recordPrice function, prices with negative exponents are not handled correctly, leading to a massive deviation in prices.

## Vulnerability Detail
Here is PythOracle#\_recordPrice function:

```solidity
    function _recordPrice(uint256 oracleVersion, PythStructs.Price memory price) private {
        _prices[oracleVersion] = Fixed6Lib.from(price.price).mul(
            Fixed6Lib.from(SafeCast.toInt256(10 ** SafeCast.toUint256(price.expo > 0 ? price.expo : -price.expo)))
        );
        _publishTimes[oracleVersion] = price.publishTime;
    }
```

If price is 5e-5 for example, it will be recorded as 5e5
If price is 5e-6, it will be recorded as 5e6.

As we can see, there is a massive deviation in recorded price from actual price whenever price's exponent is negative

## Impact
Wrong prices will be recorded.
For example,
If priceA is 5e-5, and priceB is 5e-6. But due to the wrong conversion,

- There is a massive change in price(5e5 against 5e-5)
- we know that priceA is ten times larger than priceB, but priceA will be recorded as ten times smaller than priceB.
  Unfortunately, current payoff functions may not be able to take care of these discrepancies

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L203

## Tool used

Manual Review

## Recommendation

In PythOracle.sol, `_prices` mapping should not be ` mapping(uint256 => Fixed6) private _prices;`
Instead, it should be ` mapping(uint256 => Price) private _prices;`, where Price is a struct that stores the price and expo:

```solidity
struct Price{
    Fixed6 price,
    int256 expo
}
```

This way, the price exponents will be preserved, and can be used to scale the prices correctly wherever it is used.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> h



# Issue H-3: Vault.sol: `settle`ing the 0 address will disrupt accounting 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/62 

## Found by 
Emmanuel, WATCHPUG, bin2chen
Due to the ability of anyone to settle the 0 address, the global assets and global shares will be wrong because lower keeper fees were deducted within the `_settle` function.

## Vulnerability Detail
Within `Vault#_loadContext` function, the context.global is the account of the 0 address, while context.local is the account of the address to be updated or settled:

```solidity
function _loadContext(address account) private view returns (Context memory context) {
    ...
    context.global = _accounts[address(0)].read();
    context.local = _accounts[account].read();
    context.latestCheckpoint = _checkpoints[context.global.latest].read();
}
```

If a user settles the 0 address, the global account will be updated with wrong data.

Here is the \_settle logic:

```solidity
function _settle(Context memory context) private {
    // settle global positions
    while (
        context.global.current > context.global.latest &&
        _mappings[context.global.latest + 1].read().ready(context.latestIds)
    ) {
        uint256 newLatestId = context.global.latest + 1;
        context.latestCheckpoint = _checkpoints[newLatestId].read();
        (Fixed6 collateralAtId, UFixed6 feeAtId, UFixed6 keeperAtId) = _collateralAtId(context, newLatestId);
        context.latestCheckpoint.complete(collateralAtId, feeAtId, keeperAtId);
        context.global.processGlobal(
            newLatestId,
            context.latestCheckpoint,
            context.latestCheckpoint.deposit,
            context.latestCheckpoint.redemption
        );
        _checkpoints[newLatestId].store(context.latestCheckpoint);
    }

    // settle local position
    if (
        context.local.current > context.local.latest &&
        _mappings[context.local.current].read().ready(context.latestIds)
    ) {
        uint256 newLatestId = context.local.current;
        Checkpoint memory checkpoint = _checkpoints[newLatestId].read();
        context.local.processLocal(
            newLatestId,
            checkpoint,
            context.local.deposit,
            context.local.redemption
        );
    }
}
```

If settle is called on 0 address, \_loadContext will give context.global and context.local same data.
In the \_settle logic, after the global account(0 address) is updated with the correct data in the `while` loop(specifically through the processGlobal function),
the global account gets reupdated with wrong data within the `if` statement through the processLocal function.

Wrong assets and shares will be recorded.
The global account's assets and shares should be calculated with toAssetsGlobal and toSharesGlobal respectively, but now, they are calculated with toAssetsLocal and toSharesLocal.

toAssetsGlobal subtracts the globalKeeperFees from the global deposited assets, while toAssetsLocal subtracts globalKeeperFees/Checkpoint.count fees from the local account's assets.

So in the case of settling the 0 address, where global account and local account are both 0 address, within the while loop of \_settle function, depositedAssets-globalKeeperFees is recorded for address(0), but then, in the `if` statement, depositedAssets-(globalAssets/Checkpoint.count) is recorded for address(0)

And within the `Vault#_saveContext` function, context.global is saved before context.local, so in this case, context.global(which is 0 address with correct data) is overridden with context.local(which is 0 address with wrong data).

## Impact
The global account will be updated with wrong data, that is, global assets and shares will be higher than it should be because lower keeper fees was deducted.

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L190
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L315
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Checkpoint.sol#L99
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Checkpoint.sol#L119


## Tool used

Manual Review

## Recommendation
I believe that the ability to settle the 0 address is intended, so an easy fix is to save local context before saving global context:
Before:

```solidity
    function _saveContext(Context memory context, address account) private {
        _accounts[address(0)].store(context.global);
        _accounts[account].store(context.local);
        _checkpoints[context.currentId].store(context.currentCheckpoint);
    }
```

After:

```solidity
    function _saveContext(Context memory context, address account) private {
        _accounts[account].store(context.local);
        _accounts[address(0)].store(context.global);
        _checkpoints[context.currentId].store(context.currentCheckpoint);
    }
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> h



**arjun-io**

Great find - we'll fix this

# Issue M-1: PythOracle `commit()` function doesn't require (nor stores) pyth price publish timestamp to be after the previous commit's publish timestamp, which makes it possible to manipulate price to unfairly liquidate users and possible stealing protocol funds 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/44 

## Found by 
panprog

PythOracle allows any user to commit non-requested oracle version. However, it doesn't verify pyth price publish timestamp to be in order (like `commitRequested` does). This makes it possible to commit prices out of order, potentially leading to price manipulations allowing to unfairly liquidate users or steal funds from the protocol.

## Vulnerability Detail

PythOracle `commitRequested()` has the following check:
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L138-L139

However, `commit()` doesn't have the same check. This allows malicious user to commit prices out of order, which can potentially lead to price manipulation attacks and unfair liquidation of users or loss of protocol funds.

For example, the following scenario is possible:
Timestamp = 100: Alice requests to open 1 ETH long position with $10 collateral
Timestamp = 113: pyth price = $980
Timestamp = 114: pyth price = $990
Timestamp = 115: pyth price = $1000
Timestamp = 116: Keeper commits requested price = $1000 (with publish timestamp = 115)
Timestamp = 117: Malicious user Bob commits oracle version 101 with price = $980 (publish timestamp = 113) and immediately liquidates Alice.

Even though the current price is $1000, Alice is liquidated using the price which is *earlier* than the price when Alice position is opened, which is unfair liquidation. The other more complex scenarios are also possible for malicious Bob to liquidate itself to steal protocol funds.

## Impact

Unfair liquidation as described in the scenario above or possible loss of protocol funds.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Add publish time check and store publish time in `PythOracle.commit()`:
```solidity
    function commit(uint256 oracleVersion, bytes calldata updateData) external payable {
        // Must be before the next requested version to commit, if it exists
        // Otherwise, try to commit it as the next request version to commit
        if (versionList.length > nextVersionIndexToCommit && oracleVersion >= versionList[nextVersionIndexToCommit]) {
            commitRequested(nextVersionIndexToCommit, updateData);
            return;
        }

+        if (pythPrice.publishTime <= _lastCommittedPublishTime) revert PythOracleNonIncreasingPublishTimes();
+        _lastCommittedPublishTime = pythPrice.publishTime;

        PythStructs.Price memory pythPrice = _validateAndGetPrice(oracleVersion, updateData);
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> m



# Issue M-2: During oracle provider switch, if it is impossible to commit the last request of previous provider, then the oracle will get stuck (no price updates) without any possibility to fix it 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/46 

## Found by 
panprog

When the oracle provider is updated (switched to new provider), the latest status (price) returned by the oracle will come from the previous provider until the last request is commited for it, only then the price feed from the new provider will be used. However, it can happen that it's impossible to commit the latest request: for example, if pyth signature server is down for the period it is needed, or if all keepers were down for that time period, so valid price with signature for the timestamp required is not available. In this case, the oracle price will be stuck, because it will ignore new provider, but the previous provider can never finalize (commit a fresh price). It is also impossible to cancel provider switch as there is no such function. As such, the oracle price will get stuck and will never update, breaking the whole protocol with user funds stuck in the protocol.

## Vulnerability Detail

The way oracle provider switch works is the following:
1. `Oracle.update()` is called to set a new provider. This is only allowed if there is no other provider switch pending.
2. There is a brief transition period, when both the previous provider and a new provider are active. This is to ensure that all the requests made to the previous oracle are commited before switching to a new provider. This is handled by the `Oracle._handleLatest()` function, in particular the switch to a new provider occurs only when `Oracle.latestStale()` returns true. The lines of interest to us are:
```solidity
        uint256 latestTimestamp = global.latest == 0 ? 0 : oracles[global.latest].provider.latest().timestamp;
        if (uint256(oracles[global.latest].timestamp) > latestTimestamp) return false;
```
`latestTimestamp` - is the timestamp of last commited price for the previous provider
`oracles[global.latest].timestamp` is the timestamp of the last requested price for the previous provider
The switch doesn't occur, until last commited price is equal to or after the last request timestamp for the previous provider.
3. The functions to commit the price are in PythOracle: `commitRequested` and `commit`. 
3.1. `commitRequested` requires publish timestamp of the pyth price to be within `MIN_VALID_TIME_AFTER_VERSION`..`MAX_VALID_TIME_AFTER_VERSION` from *request time*. It is possible that pyth price with signature in this time period is not available for different reasons (pyth price feed is down, keeper was down during this period and didn't collect price and signature):
```solidity
        uint256 versionToCommit = versionList[versionIndex];
        PythStructs.Price memory pythPrice = _validateAndGetPrice(versionToCommit, updateData);
```
`versionList` is an array of oracle request timestamps. And `_validateAndGetPrice()` filters the price within the interval specified (if it is not in the interval, it will revert):
```solidity
        return pyth.parsePriceFeedUpdates{value: pyth.getUpdateFee(updateDataList)}(
            updateDataList,
            idList,
            SafeCast.toUint64(oracleVersion + MIN_VALID_TIME_AFTER_VERSION),
            SafeCast.toUint64(oracleVersion + MAX_VALID_TIME_AFTER_VERSION)
        )[0].price;
```
3.2. `commit` can not be done with timestamp older than the first oracle request timestamp: if any oracle request is still active, it will simply redirect to `commitRequested`:
```solidity
        if (versionList.length > nextVersionIndexToCommit && oracleVersion >= versionList[nextVersionIndexToCommit]) {
            commitRequested(nextVersionIndexToCommit, updateData);
            return;
        }
```
4. All new oracle requests are directed to a **new** provider, this means that previous provider can not receive any new requests (which allows to finalize it):
```solidity
    function request(address account) external onlyAuthorized {
        (OracleVersion memory latestVersion, uint256 currentTimestamp) = oracles[global.current].provider.status();

        oracles[global.current].provider.request(account);
        oracles[global.current].timestamp = uint96(currentTimestamp);
        _updateLatest(latestVersion);
    }
```

So the following scenario is possible:
timestamp=69: oracle price is commited for timestamp=50
timestamp=70: user requests to open position (`Oracle.request()` is made)
timestamp=80: owner calls `Oracle.update()`
timestamp=81: pyth price signing service goes offline (or keeper goes offline)
...
timestamp=120: signing service goes online again.
timestamp=121: another user requests to open position (`Oracle.request()` is made, directed to new provider)
timestamp=200: new provider's price is commited (`commitRequested` is called with timestamp=121)

At this time, `Oracle.latest()` will return price at timestamp=50. It will ignore new provider's latest commit, because previous provider last request (timestamp=70) is still not commited. Any new price requests and commits to a new provider will be ignored, but the previous provider can not be commited due to absence of prices in the valid time range. It is also not possible to change oracle for the market, because there is no such function. It is also impossible to cancel provider update and impossible to change the provider back to previous one, as all of these will revert.

It is still possible for the owner to manually whitelist some address to call `request()` for the previous provider. However, this situation provides even worse result. While the latest version for the previous provider will now be later than the last request, so it will let the oracle switch to new provider, however `oracle.status()` will briefly return invalid oracle version, because it will return oracle version at the timestamp = last request before the provider switch, which will be invalid (the new request will be after that timestamp):

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/Oracle.sol#L103

This can be abused by some user who can backrun the previous provider oracle commit (or commit himself) and use the invalid oracle returned by `status()` (oracle version with price = 0). Market doesn't expect the oracle status to return invalid price (it is expected to be always valid), so it will use this invalid price as if it's a normal price = 0, which will totally break the market:

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L574-L577

So if the oracle provider switch becomes stuck, there is no way out and the market will become stale, not allowing any user to withdraw the funds.

## Impact

Switching oracle provider can make the oracle stuck and stop updating new prices. This will mean the market will become stale and will revert on all requests from user, disallowing to withdraw funds, bricking the contract entirely.

## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/Oracle.sol#L112-L113

## Tool used

Manual Review

## Recommendation

There are multiple possible ways to fix this. For example, allow to finalize previous provider if the latest commit from the new provider is newer than the latest commit from the previous provider by `GRACE_PERIOD` seconds. Or allow PythOracle to `commit` directly (instead of via `commitRequested`) if the commit oracleVersion is newer than the last request by `GRACE_PERIOD` seconds.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> m



# Issue M-3: Invalid oracle versions can cause desync of global and local positions making protocol lose funds and being unable to pay back all users 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/49 

## Found by 
panprog

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



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> m



# Issue M-4: Protocol fee from Market.sol is locked 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/52 

## Found by 
0x73696d616f, Emmanuel, WATCHPUG, bin2chen

The `MarketFactory#fund` calls the specified market's `Market#claimFee` function.
This will send the protocolFee to the MarketFactory contract.
MarketFactory contract does not max approve any address to spend its tokens, and there is no function that can be used to get the funds out of the contract, so the funds are permanently locked in MarketFactory.

## Vulnerability Detail
Here is `MarketFactory#fund` function:

```solidity
  function fund(IMarket market) external {
        if (!instances(IInstance(address(market)))) revert FactoryNotInstanceError();
@>      market.claimFee();
    }
```

This is `Market#claimFee` function:

```solidity
    function claimFee() external {
        Global memory newGlobal = _global.read();

        if (_claimFee(address(factory()), newGlobal.protocolFee)) newGlobal.protocolFee = UFixed6Lib.ZERO;
        ...
    }
```

This is the internal `_claimFee` function:

```solidity
    function _claimFee(address receiver, UFixed6 fee) private returns (bool) {
        if (msg.sender != receiver) return false;

        token.push(receiver, UFixed18Lib.from(fee));
        emit FeeClaimed(receiver, fee);
        return true;
    }
```

As we can see, when `MarketFactory#fund` is called, Market#claimFee gets called which will send the protocolFee to msg.sender(MarketFacttory).
When you check through the MarketFactory contract, there is no place where another address(such as protocol multisig, treasury or an EOA) is approved to spend MarketFactory's funds, and also, there is no function in the contract that can be used to transfer MarketFactory's funds.
This causes locking of the protocol fees.

## Impact
Protocol fees cannot be withdrawn
## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/MarketFactory.sol#L89

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L133

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L145-L151

## Tool used

Manual Review

## Recommendation
Consider adding a `withdraw` function that protocol can use to get the protocolFee out of the contract.
You can have the withdraw function transfer the MarketFactory balance to the treasury or something.



## Discussion

**sherlock-admin**

2 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> h

**n33k** commented:
> medium



**arjun-io**

We originally wanted to keep the funds in the Factory (for a future upgrade) but it might make sense to instead allow the Factory Owner (Timelock) to claim these funds instead

# Issue M-5: Bad debt (shortfall) liquidation leaves liquidated user in a negative collateral balance which can cause bank run and loss of funds for the last users to withdraw 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/72 

## Found by 
WATCHPUG, panprog

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



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> m



**arjun-io**

While we like the recommended approach it might be overly cumbersome to implement and still does not fully prevent the shortfall bank run situation. Bad debt/shortfall is necessary in Perennial and should be thoroughly minimized through correct parameter setting.

# Issue M-6: It is possible to open and liquidate your own position in 1 transaction to overcome efficiency and liquidity removal limits at almost no cost 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/104 

## Found by 
Emmanuel, panprog

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



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> m



**arjun-io**

The self liquidations do seem possible here, we'll look further into the downstream impacts to figure out any fixes we want to implement.

# Issue M-7: update() wrong privilege control 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/121 

## Found by 
bin2chen
`oracle.update()`  wrong privilege control
lead to `OracleFactory.update()` unable to add `oracleProvider`

## Vulnerability Detail
in `OracleFactory.update()` will call `oracle.update()`

```solidity
contract OracleFactory is IOracleFactory, Factory {
...
    function update(bytes32 id, IOracleProviderFactory factory) external onlyOwner {
        if (!factories[factory]) revert OracleFactoryNotRegisteredError();
        if (oracles[id] == IOracleProvider(address(0))) revert OracleFactoryNotCreatedError();

        IOracleProvider oracleProvider = factory.oracles(id);
        if (oracleProvider == IOracleProvider(address(0))) revert OracleFactoryInvalidIdError();

        IOracle oracle = IOracle(address(oracles[id]));
@>      oracle.update(oracleProvider);
    }

```

But `oracle.update()` permission is needed for `OracleFactory.owner()` and not `OracleFactory` itself.

```solidity
@>  function update(IOracleProvider newProvider) external onlyOwner {
        _updateCurrent(newProvider);
        _updateLatest(newProvider.latest());
    }

    modifier onlyOwner {
@>      if (msg.sender != factory().owner()) revert InstanceNotOwnerError(msg.sender);
        _;
    }
```

This results in `OracleFactory` not being able to do `update()`.
Suggest changing the limit of ``oracle.update()`` to ``factory()``.

## Impact

`OracleFactory.update()` unable to add `IOracleProvider`


## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/OracleFactory.sol#L81



## Tool used

Manual Review

## Recommendation

```solidity
contract Oracle is IOracle, Instance {
...

-   function update(IOracleProvider newProvider) external onlyOwner {
+   function update(IOracleProvider newProvider) external {
+       require(msg.sender == factory(),"invalid sender");
        _updateCurrent(newProvider);
        _updateLatest(newProvider.latest());
    }
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> m



# Issue M-8: `_accumulateFunding()` maker will get the wrong amount of funding fee. 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/139 

## Found by 
WATCHPUG

## Vulnerability Detail

The formula that calculates the amount of funding in `Version#_accumulateFunding()` on the maker side is incorrect. This leads to an incorrect distribution of funding between the minor and the maker's side.

```solidity
// Redirect net portion of minor's side to maker
if (fromPosition.long.gt(fromPosition.short)) {
    fundingValues.fundingMaker = fundingValues.fundingShort.mul(Fixed6Lib.from(fromPosition.skew().abs()));
    fundingValues.fundingShort = fundingValues.fundingShort.sub(fundingValues.fundingMaker);
}
if (fromPosition.short.gt(fromPosition.long)) {
    fundingValues.fundingMaker = fundingValues.fundingLong.mul(Fixed6Lib.from(fromPosition.skew().abs()));
    fundingValues.fundingLong = fundingValues.fundingLong.sub(fundingValues.fundingMaker);
}
```

## PoC

Given:

- long/major: 1000
- short/minor: 1
- maker: 1

Then:

1. skew(): 999/1000
2. fundingMaker: 0.999 of the funding
3. fundingShort: 0.001 of the funding

While the maker only matches for `1` of the major part and contributes to half of the total short side, it takes the entire funding.

## Impact

## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/types/Version.sol#L207-L215


## Tool used

Manual Review

## Recommendation

The correct formula to calculate the amount of funding belonging to the maker side should be:

```markdown
fundingMakerRatio = min(maker, major - minor) / min(major, minor + maker)
fundingMaker = fundingMakerRatio * fundingMinor
```



## Discussion

**sherlock-admin**

2 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> m

**panprog** commented:
> medium because incorrect result only starts appearing if abs(long-short) > maker and the larger the difference, the more incorrect the split of funding is. But this situation is exceptional case, most of the time abs(long-short) < maker due to efficiency and liquidity limits



**arjun-io**

We'd like to re-open this as it does appear to be a valid issue. Medium severity seems correct here

# Issue M-9: Early Vault depositor can manipulate exchange rates to steal funds from later depositors 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/154 

## Found by 
tives

An early user can manipulate the price per share and profit from late users' deposits because of the precision loss caused by the large value of price per share.****

## Vulnerability Detail

### Part 1: deposit

A malicious early user can deposit `1 wei` of token as the first depositor to the Vault, and get `1 wei` of shares.

User shares are set in processLocal/processGlobal. These are called in _settle, which is also called in `Vault.update(... depositAssets ...)` 

```solidity
function update(
        address account,
        UFixed6 depositAssets,
        UFixed6 redeemShares,
        UFixed6 claimAssets
    ) external whenNotPaused {
        _settleUnderlying();
        Context memory context = _loadContext(account);

        _settle(context);

function _settle()
...
     context.local.processLocal(

```

```solidity
function processLocal(
    Account memory self,
    uint256 latestId,
    Checkpoint memory checkpoint,
    UFixed6 deposit,
    UFixed6 redemption
) internal pure {
    self.latest = latestId;
    (self.assets, self.shares) = (
        self.assets.add(checkpoint.toAssetsLocal(redemption)),
        self.shares.add(checkpoint.toSharesLocal(deposit))
    );

_toSharesLocal > _toShares > _withSpread

function _withSpread(Checkpoint memory self, UFixed6 amount) private pure returns (UFixed6) {
    UFixed6 selfAssets = UFixed6Lib.from(self.assets.max(Fixed6Lib.ZERO));
    UFixed6 totalAmount = self.deposit.add(self.redemption.muldiv(selfAssets, self.shares));

    return totalAmount.isZero() ?
        amount :
        amount.muldiv(totalAmount.sub(self.fee.min(totalAmount)), totalAmount);
}
```

`_withSpread` sets the shares amount to 1 wei  (`return totalAmount.isZero() ? amount : …`)

Now, the attacker has 1 share for 1 wei of tokens.

### Part 2: share value inflation

Then the attacker can send `10000e18 - 1` of `asset` token to the Vault and inflate the price per share from 1.0000 to an extreme value of 1.0000e22 ( from `(1 + 10000e18 - 1) / 1`) .

### Part 3: redeeming for inflated share price

The `_socialize` function returns the claim amount. `_socialize` in turn calls `_collateral`, which uses `asset.balanceOf()` to return the collateral amount

```solidity
_socialize() returns (UFixed6 claimAmount) >  _collateral

function _collateral(Context memory context) public view returns (Fixed6 value) {
    value = Fixed6Lib.from(UFixed6Lib.from(asset.balanceOf()));
    for (uint256 marketId; marketId < context.markets.length; marketId++)
        value = value.add(context.markets[marketId].collateral);
}
```

This means that the `claimAmount` is calculated according to asset.balanceOf(). This means that adversary has retained her initial shares but inflated the share price via sending in tokens manually.

She can now receive more tokens than her shares are worth. These tokens come from the later depositors.

## Impact

Initial depositor steals from late depositors due to inflated share price.

## Code Snippet

```solidity
/// @notice Returns the real amount of collateral in the vault
/// @return value The real amount of collateral in the vault
function _collateral(Context memory context) public view returns (Fixed6 value) {
    value = Fixed6Lib.from(UFixed6Lib.from(asset.balanceOf()));
    for (uint256 marketId; marketId < context.markets.length; marketId++)
        value = value.add(context.markets[marketId].collateral);
}
```

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol/#L486

## Tool used

Manual Review

## Recommendation

Require a bigger initial deposit or premint some shares and burn them before the first deposit. This is a well known attack vector for ERC4626 vaults. You can read more about it [here](https://github.com/transmissions11/solmate/issues/178)



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> m



# Issue M-10: Oracle requests dont check if latest provider is still active 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/169 

## Found by 
minhtrng

Requests in the oracle will always request from the `current` provider even if the `latest` provider is still active. This can cause a gap in price, when `latest` provider will not be updated but its most recent data will still be used.

## Vulnerability Detail

`Oracle.request` always requests from `global.current`:

```js
oracles[global.current].provider.request(account);
```

If the `global.latest` is not stale yet, its `latest()` data will be used (but not updated, due to the above):

```js
function _handleLatest(
    OracleVersion memory currentOracleLatestVersion
) private view returns (OracleVersion memory latestVersion) {
    if (global.current == global.latest) return currentOracleLatestVersion;

    bool isLatestStale = _latestStale(currentOracleLatestVersion);
    latestVersion = isLatestStale ? currentOracleLatestVersion : oracles[global.latest].provider.latest();

    uint256 latestOracleTimestamp =
        uint256(isLatestStale ? oracles[global.current].timestamp : oracles[global.latest].timestamp);
        
    if (!isLatestStale && latestVersion.timestamp > latestOracleTimestamp)
        return at(latestOracleTimestamp);
    }
```

## Impact

Stale price being used due to old provider still being active but receiving no udpate for the last granularity window that it is being active.

## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/c2f6141c231d3952898ea47793872cac69a1d2af/perennial-v2/packages/perennial-oracle/contracts/Oracle.sol#L38

## Tool used

Manual Review

## Recommendation

Request from `oracles[global.latest].provider` if `block.timestamp` is not past its ending time yet.



## Discussion

**sherlock-admin**

2 comment(s) were left on this issue during the judging contest.

**__141345__** commented:
> m

**panprog** commented:
> invalid because no real explanation of the issue



# Issue M-11: [Perennial Self Report] Fix non-requested commits after oracle grace period 

Source: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/177 

## Found by 
Protocol Team
Medium

When a requested version was unavailable, non-requested versions would be blocked from being to be committed until a new requested version was committed. This could prevent liquidations from occurring.

https://github.com/equilibria-xyz/perennial-v2/pull/58



## Discussion

**hrishibhat**

This issue is not included in the contest pool rewards

