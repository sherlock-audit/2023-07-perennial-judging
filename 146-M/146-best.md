Funny Pickle Cottonmouth

medium

# No L2 sequencer check when getting ETH price for the sake of calculating keeper fees

The lack of a sequencer uptime check allows for the use of a stale price for calculating keeper payouts.

## Vulnerability Detail

In the case of sequencer downtime, the prices provided by the Chainlink ETH price feed can become outdated, paying the keepers out with outdated rates.

## Impact

If ETH was to significantly move in price while the sequencer is down it will cause financial damage to the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/root/contracts/attribute/Kept.sol#L62

```solidity
function _etherPrice() private view returns (UFixed18) {
  // @audit no L2 sequencer uptime check
  (, int256 answer, , ,) = ethTokenOracleFeed().latestRoundData();
	return UFixed18Lib.from(Fixed18Lib.ratio(answer, 1e8)); // chainlink eth-usd feed uses 8 decimals
}
```

## Tool used

Manual Review

## Recommendation

Consider using the following Chainlink feed to conduct a check for whether the L2 sequencer is down or not.

https://blog.chain.link/how-to-use-chainlink-price-feeds-on-arbitrum/#almost_done!_meet_the_l2_sequencer_health_flag