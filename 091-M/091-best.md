Rich Lilac Viper

high

# Keepers will suffer significant losses due to miss compensation for L1 rollup fees
While keepers submits transactions to L2 EVM chains, they need to pay both L2 execution fee and L1 rollup fee. Actually, L1 fees are much higher than L2 fees. In many case, L2 fees can be practically negligible. The current implementation only compensate and incentive keepers based on L2 gas consumption, keepers will suffer significant losses.

## Vulnerability Detail
As shown of ````keep()```` modifier (L40-57), only L2  execution fee are compensated.
```solidity
File: perennial-v2\packages\perennial-oracle\contracts\pyth\PythOracle.sol
124:     function commitRequested(uint256 versionIndex, bytes calldata updateData)
125:         public
126:         payable
127:         keep(KEEPER_REWARD_PREMIUM, KEEPER_BUFFER, "")
128:     {
...
157:     }

File: perennial-v2\packages\perennial-extensions\contracts\MultiInvoker.sol
359:     function _executeOrder(
360:         address account,
361:         IMarket market,
362:         uint256 nonce
363:     ) internal keep (
364:         UFixed18Lib.from(keeperMultiplier),
365:         GAS_BUFFER,
366:         abi.encode(account, market, orders(account, market, nonce).fee)
367:     ) {
...
384:     }


File: root\contracts\attribute\Kept.sol
40:     modifier keep(UFixed18 multiplier, uint256 buffer, bytes memory data) {
41:         uint256 startGas = gasleft();
42: 
43:         _;
44: 
45:         uint256 gasUsed = startGas - gasleft();
46:         UFixed18 keeperFee = UFixed18Lib.from(gasUsed)
47:             .mul(multiplier)
48:             .add(UFixed18Lib.from(buffer))
49:             .mul(_etherPrice())
50:             .mul(UFixed18.wrap(block.basefee));
51: 
52:         _raiseKeeperFee(keeperFee, data);
53: 
54:         keeperToken().push(msg.sender, keeperFee);
55: 
56:         emit KeeperCall(msg.sender, gasUsed, multiplier, buffer, keeperFee);
57:     }

```

Takes a random selected transaction at the writing time
https://optimistic.etherscan.io/tx/0xbb8e68e21c92acf4171fb6041b758b55acc3c559ec4595ed1129d534d90de995

We can find the L1 fee is much expensive than L2 fee.
```solidity
L2 fee = 0.06 Gwei * 113,449 ~= 6,806 Gwei
L1 fee = 0.00006565634612417 ETH ~= 65656 Gwei
L1 / L2 = 964%
```

Typically, L1 fees are dynamic and determined by the calldata length and smoothed Ethereum gas prices. To submit Pyth oracle price, the ````VAA```` calldata will be 1700+ bytes, keepers need to pay much L1 rollup fee.

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/test/integration/pyth/PythOracle.test.ts#L34

## Impact
No enough incentive for keeper to submit oracle price and execute orders, the system will not work.

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/root/contracts/attribute/Kept.sol#L40

## Tool used

Manual Review

## Recommendation
Compensating  L1 rollup fee, here are some reference:
https://docs.arbitrum.io/arbos/l1-pricing
https://community.optimism.io/docs/developers/build/transaction-fees/#the-l1-data-fee