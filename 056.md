Fresh Aegean Sparrow

medium

# PythOracle:if price.expo is less than 0, wrong prices will be recorded
## Summary
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
