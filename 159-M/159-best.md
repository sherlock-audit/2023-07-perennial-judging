Original Tiger Panther

medium

# Missing `updatedAt` and recommended timeout checks in `Kept.sol` fetched chainlink prices
## Summary
[`PythOracle`](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L127) incentivizes the `keeper` with an amount pro-rata to the ether price, fetched from a Chainlink oracle. When using chainlink prices, it's important to check that the `updatedAt` return value from the `latestRoundData()` call is different than 0. Additionaly, a timeout should be added after which the `updatedAt` value is no longer valid (not fresh enough).

## Vulnerability Detail
[Here](https://0xmacro.com/blog/how-to-consume-chainlink-price-feeds-safely/) is a great article from 0xmacro explaining in detail these 2 important measures. The `updatedAt` value should be different than 0 and smaller than the current timestamp by only a hardcode timeout.

## Impact
The `keeper` is incorrectly incentivized and could incur in losses. Would place the system in a DoS state as no one would want to incur losses to update the oracle prices (or losses to the protocol team).

## Code Snippet
In [`Kept.sol`](https://github.com/sherlock-audit/2023-07-perennial/blob/main/root/contracts/attribute/Kept.sol#L40-L64), notice the `keep()` modifier and `_etherPrice()` functions. There are no checks in the Chainlink answer (besides it being negative, as it would underflow when converted to `UFixed18`).
```solidity
modifier keep(UFixed18 multiplier, uint256 buffer, bytes memory data) {
    uint256 startGas = gasleft();

    _;

    uint256 gasUsed = startGas - gasleft();
    UFixed18 keeperFee = UFixed18Lib.from(gasUsed)
        .mul(multiplier)
        .add(UFixed18Lib.from(buffer))
        .mul(_etherPrice())
        .mul(UFixed18.wrap(block.basefee));

    _raiseKeeperFee(keeperFee, data);

    keeperToken().push(msg.sender, keeperFee);

    emit KeeperCall(msg.sender, gasUsed, multiplier, buffer, keeperFee);
}

/// @notice Returns the price of ETH in terms of the keeper token
/// @return The price of ETH in terms of the keeper token
function _etherPrice() private view returns (UFixed18) {
    (, int256 answer, , ,) = ethTokenOracleFeed().latestRoundData();
    return UFixed18Lib.from(Fixed18Lib.ratio(answer, 1e8)); // chainlink eth-usd feed uses 8 decimals
}
```

## Tool used
Vscode, Hardhat, Manual Review

## Recommendation
Add the following checks to `_etherPrice()`:
```solidity
function _etherPrice() private view returns (UFixed18) { // TO SUBMIT missing validation
    (, int256 answer, , uint256 updatedAt,) = ethTokenOracleFeed().latestRoundData();
    if (updatedAt == 0) revert OracleNotUpdatedError(); // sanity check
    if (block.timestamp - updatedAt > TIMEOUT) revert OracleStalePriceError(); // price staleness check
    return UFixed18Lib.from(Fixed18Lib.ratio(answer, 1e8)); // chainlink eth-usd feed uses 8 decimals
}
```
