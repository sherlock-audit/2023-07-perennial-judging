Itchy Licorice Stork

medium

# Oracle requests dont check if latest provider is still active
## Summary

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