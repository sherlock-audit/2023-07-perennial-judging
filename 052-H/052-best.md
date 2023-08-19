Fresh Aegean Sparrow

high

# Protocol fee from Market.sol is locked

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