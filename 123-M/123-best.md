Savory Laurel Gazelle

medium

# keep() may not be able to execute
`Kept.keep()` calls `_raiseKeeperFee()` to get the token
Most current implementations of `_raiseKeeperFee()` remove the trailing number `18 -> 6 -> 18`.
However, when `Kept.keep()` is transferred to `msg.sender`, the trailing number is still included, which may leads to transfer failure

## Vulnerability Detail
`Kept.keep()` calls `_raiseKeeperFee()` to get the token
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
@>      _raiseKeeperFee(keeperFee, data);

           keeperToken().push(msg.sender, keeperFee);
```

Currently both `PythOracle` and `MultiInvoker` implement the `_raiseKeeperFee()` method.

Let's take `MultiInvoker_raiseKeeperFee()` for example

```solidity
contract MultiInvoker is IMultiInvoker, Kept {
...
    function _raiseKeeperFee(UFixed18 keeperFee, bytes memory data) internal override {
        (address account, address market, UFixed6 fee) = abi.decode(data, (address, address, UFixed6));
        if (keeperFee.gt(UFixed18Lib.from(fee))) revert MultiInvokerMaxFeeExceededError();

        IMarket(market).update(
            account,
            UFixed6Lib.MAX,
            UFixed6Lib.MAX,
            UFixed6Lib.MAX,
@>          Fixed6Lib.from(Fixed18Lib.from(-1, keeperFee), true),
            false
        );

    }

contract Market is IMarket, Instance, ReentrancyGuard {
...
    function _update(
        Context memory context,
        address account,
        UFixed6 newMaker,
        UFixed6 newLong,
        UFixed6 newShort,
        Fixed6 collateral,
        bool protect
    ) private {
..
        if (collateral.sign() == 1) token.pull(msg.sender, UFixed18Lib.from(collateral.abs()));
@>      if (collateral.sign() == -1) token.push(msg.sender, UFixed18Lib.from(collateral.abs()));

        // events
        emit Updated(account, context.currentTimestamp, newMaker, newLong, newShort, collateral, protect);
    }
```

We can see that `_raiseKeeperFee()` does an interception of the decimal places, removing the ones after 6. `18 -> 6 -> 18`.

But `keep()` still takes the full number of decimal places.
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

@>       keeperToken().push(msg.sender, keeperFee);
```

This causes `_raiseKeeperFee()` to get the number of tokens that may be less than ` keeperFee`
causing ` keeperToken().push(msg.sender, keeperFee);` to fail


## Impact

`keep()` may not perform properly

## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/root/contracts/attribute/Kept.sol#L54C1-L54C1

## Tool used

Manual Review

## Recommendation

`Keep()` removes decimals after the 6th digit, as usual.

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
+       keeperFee = UFixed18Lib.from(UFixed6Lib.from(keeperFee,true));
        _raiseKeeperFee(keeperFee, data);

        keeperToken().push(msg.sender, keeperFee);

        emit KeeperCall(msg.sender, gasUsed, multiplier, buffer, keeperFee);
    }
```