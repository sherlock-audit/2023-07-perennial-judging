Fresh Aegean Sparrow

medium

# Excess native value is not sent back to keeper after a commit call

PythOracle#commit requires that user sends higher than needed native value to ensure that transaction does not revert(because the amount of native value to be sent with the parsePriceFeedUpdates call is not known beforehand).
The excess native value sent by keeper is not refunded to him after the call, leading to locking of native value in PythOracle contract.

## Vulnerability Detail
When committing prices, pyth.parsePriceFeedUpdates is called, entering a native value which is determined via pyth.getUpdateFee.

```solidity
function _validateAndGetPrice(
    uint256 oracleVersion,
    bytes calldata updateData
) private returns (PythStructs.Price memory price) {
    bytes[] memory updateDataList = new bytes[](1);
    updateDataList[0] = updateData;
    bytes32[] memory idList = new bytes32[](1);
    idList[0] = id;

    return
        pyth
@>      .parsePriceFeedUpdates{ value: pyth.getUpdateFee(updateDataList) }(
            updateDataList,
            idList,
            SafeCast.toUint64(oracleVersion + MIN_VALID_TIME_AFTER_VERSION),
            SafeCast.toUint64(oracleVersion + MAX_VALID_TIME_AFTER_VERSION)
        )[0].price;
}
```

Since the native value is not constant, or known beforehand, keeper that wants to commit price has to send a higher amount of native value to ensure that transaction executes without reverting.
The problem is that, the excess native value sent by the keeper is noever refunded back to him. Instead, it remains locked in PythOracle.sol contract.

## Impact

Native value will get locked in PythOracle contract after each price commitment.

## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L190

## Tool used

Manual Review

## Recommendation

Within `_validateAndGetPrice` function, cache the update fee required for the call.
At the end of the function, refund `msg.value-cachedUpdateFee` to the keeper.
To prevent reentrancy possibilities, increment the `ethBalanceOf` for the keeper instead of refunding directly, and provide a `withdrawExcessETH` function for the keeper to withdraw his excess ETH, which would update `ethBalanceOf` that keeper to 0.