Flat Iris Rhino

medium

# Chainlink oracle will return the wrong price if the aggregator hits minAnswer
Chainlink oracle will return the wrong price if the aggregator hits minAnswer

## Vulnerability Detail
## Impact
Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price band.
The result is that if an asset experiences a huge drop in value (i.e. LUNA crash) the price of the oracle will continue to return the minPrice instead of the actual price of the asset.
This would allow user to continue borrowing with the asset but at the wrong price. [This is exactly what happened to Venus on BSC when LUNA imploded](https://rekt.news/venus-blizz-rekt/)

```Solidity

    function _etherPrice() private view returns (UFixed18) {
        (, int256 answer, , ,) = ethTokenOracleFeed().latestRoundData();
        return UFixed18Lib.from(Fixed18Lib.ratio(answer, 1e8)); // chainlink eth-usd feed uses 8 decimals
    }
```

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/root/contracts/attribute/Kept.sol#L62

## Tool used
Manual Review

## Recommendation
Consider using the following checks.

For example:

```Solidity
(uint80, int256 answer, uint, uint, uint80) = oracle.latestRoundData();

// minPrice check
require(answer > minPrice, "Min price exceeded");
// maxPrice check
require(answer < maxPrice, "Max price exceeded");
```