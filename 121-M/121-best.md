Savory Laurel Gazelle

medium

# update() wrong privilege control
## Summary
`oracle.update()`  wrong privilege control
lead to `OracleFactory.update()` unable to add `oracleProvider`

## Vulnerability Detail
in `OracleFactory.update()` will call `oracle.update()`

```solidity
contract OracleFactory is IOracleFactory, Factory {
...
    function update(bytes32 id, IOracleProviderFactory factory) external onlyOwner {
        if (!factories[factory]) revert OracleFactoryNotRegisteredError();
        if (oracles[id] == IOracleProvider(address(0))) revert OracleFactoryNotCreatedError();

        IOracleProvider oracleProvider = factory.oracles(id);
        if (oracleProvider == IOracleProvider(address(0))) revert OracleFactoryInvalidIdError();

        IOracle oracle = IOracle(address(oracles[id]));
@>      oracle.update(oracleProvider);
    }

```

But `oracle.update()` permission is needed for `OracleFactory.owner()` and not `OracleFactory` itself.

```solidity
@>  function update(IOracleProvider newProvider) external onlyOwner {
        _updateCurrent(newProvider);
        _updateLatest(newProvider.latest());
    }

    modifier onlyOwner {
@>      if (msg.sender != factory().owner()) revert InstanceNotOwnerError(msg.sender);
        _;
    }
```

This results in `OracleFactory` not being able to do `update()`.
Suggest changing the limit of ``oracle.update()`` to ``factory()``.

## Impact

`OracleFactory.update()` unable to add `IOracleProvider`


## Code Snippet

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/OracleFactory.sol#L81



## Tool used

Manual Review

## Recommendation

```solidity
contract Oracle is IOracle, Instance {
...

-   function update(IOracleProvider newProvider) external onlyOwner {
+   function update(IOracleProvider newProvider) external {
+       require(msg.sender == factory(),"invalid sender");
        _updateCurrent(newProvider);
        _updateLatest(newProvider.latest());
    }
```
