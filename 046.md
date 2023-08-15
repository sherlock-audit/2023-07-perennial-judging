Wonderful Silver Loris

medium

# During oracle provider switch, if it is impossible to commit the last request of previous provider, then the oracle will get stuck (no price updates) without any possibility to fix it
## Summary

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