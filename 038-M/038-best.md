Sour Macaroon Boar

medium

# Chainlink's latestRoundData return stale or incorrect result

Medium

# Chainlink's `latestRoundData` return stale or incorrect result

## Summary
The Chainlink price feed can return a value of 0, and there is no check in the code to prevent this from happening. This could result in incorrect prices being used, which could have negative downstream impacts.

## Vulnerability Detail
The vulnerability is in the `root\contracts\attribute\Kept.sol` contract. This contract uses the `latestRoundData` function from the Chainlink price feed to get the price of ETH in terms of the keeper token. However, there is no check to ensure that the answer returned is not 0. If the answer is 0, then an incorrect price will be used.

This incorrect price could then be used in other contracts, which could lead to problems, such as overpaying for assets or undervaluing assets.
```solidity
    /// @notice Returns the price of ETH in terms of the keeper token
    /// @return The price of ETH in terms of the keeper token
    function _etherPrice() private view returns (UFixed18) {
        (, int256 answer, , ,) = ethTokenOracleFeed().latestRoundData();
        return UFixed18Lib.from(Fixed18Lib.ratio(answer, 1e8)); // chainlink eth-usd feed uses 8 decimals
    }
```

This `_etherPrice` is used in `keep` modifier of [Kept.sol](https://github.com/sherlock-audit/2023-07-perennial/blob/main/root/contracts/attribute/Kept.sol#L36-L57) which is used in `_executeOrder` function at [perennial-v2\packages\perennial-extensions\contracts\MultiInvoker.sol](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L353-L384) and `commitRequested` function at [perennial-v2\packages\perennial-oracle\contracts\pyth\PythOracle.sol](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/pyth/PythOracle.sol#L120-L157)

## Impact
The impact of this vulnerability could be significant. If an incorrect price is used, it could lead to financial losses for users of the affected contracts. Additionally, it could damage the reputation of the Perennial protocol.

This could lead to stale prices according to the Chainlink documentation:
https://docs.chain.link/data-feeds/price-feeds/historical-data

Related report(s): 
- [rvierdiiev at IronBank](https://github.com/sherlock-audit/2023-05-ironbank-judging/issues/9)
- [oboront at Blueberry](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/128)
- [volodya at JOJO](https://github.com/sherlock-audit/2023-04-jojo-judging/issues/14)

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/root/contracts/attribute/Kept.sol#L59-L64

## Tool used
Manual Review

## Recommendation
The recommended mitigation for this vulnerability is to add a check to the `_etherPrice` function to ensure that the answer returned is not 0. This check could be implemented as follows:

```solidity
require(answer > 0, "Chainlink answer reporting 0");
```

This check will ensure that the price returned by the Chainlink price feed is always greater than 0. This will help to prevent incorrect prices from being used and mitigate the impact of this vulnerability.