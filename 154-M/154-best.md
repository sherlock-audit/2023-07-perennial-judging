Expert Lava Chinchilla

medium

# Early Vault depositor can manipulate exchange rates to steal funds from later depositors

An early user can manipulate the price per share and profit from late users' deposits because of the precision loss caused by the large value of price per share.****

## Vulnerability Detail

### Part 1: deposit

A malicious early user can deposit `1 wei` of token as the first depositor to the Vault, and get `1 wei` of shares.

User shares are set in processLocal/processGlobal. These are called in _settle, which is also called in `Vault.update(... depositAssets ...)` 

```solidity
function update(
        address account,
        UFixed6 depositAssets,
        UFixed6 redeemShares,
        UFixed6 claimAssets
    ) external whenNotPaused {
        _settleUnderlying();
        Context memory context = _loadContext(account);

        _settle(context);

function _settle()
...
     context.local.processLocal(

```

```solidity
function processLocal(
    Account memory self,
    uint256 latestId,
    Checkpoint memory checkpoint,
    UFixed6 deposit,
    UFixed6 redemption
) internal pure {
    self.latest = latestId;
    (self.assets, self.shares) = (
        self.assets.add(checkpoint.toAssetsLocal(redemption)),
        self.shares.add(checkpoint.toSharesLocal(deposit))
    );

_toSharesLocal > _toShares > _withSpread

function _withSpread(Checkpoint memory self, UFixed6 amount) private pure returns (UFixed6) {
    UFixed6 selfAssets = UFixed6Lib.from(self.assets.max(Fixed6Lib.ZERO));
    UFixed6 totalAmount = self.deposit.add(self.redemption.muldiv(selfAssets, self.shares));

    return totalAmount.isZero() ?
        amount :
        amount.muldiv(totalAmount.sub(self.fee.min(totalAmount)), totalAmount);
}
```

`_withSpread` sets the shares amount to 1 wei  (`return totalAmount.isZero() ? amount : …`)

Now, the attacker has 1 share for 1 wei of tokens.

### Part 2: share value inflation

Then the attacker can send `10000e18 - 1` of `asset` token to the Vault and inflate the price per share from 1.0000 to an extreme value of 1.0000e22 ( from `(1 + 10000e18 - 1) / 1`) .

### Part 3: redeeming for inflated share price

The `_socialize` function returns the claim amount. `_socialize` in turn calls `_collateral`, which uses `asset.balanceOf()` to return the collateral amount

```solidity
_socialize() returns (UFixed6 claimAmount) >  _collateral

function _collateral(Context memory context) public view returns (Fixed6 value) {
    value = Fixed6Lib.from(UFixed6Lib.from(asset.balanceOf()));
    for (uint256 marketId; marketId < context.markets.length; marketId++)
        value = value.add(context.markets[marketId].collateral);
}
```

This means that the `claimAmount` is calculated according to asset.balanceOf(). This means that adversary has retained her initial shares but inflated the share price via sending in tokens manually.

She can now receive more tokens than her shares are worth. These tokens come from the later depositors.

## Impact

Initial depositor steals from late depositors due to inflated share price.

## Code Snippet

```solidity
/// @notice Returns the real amount of collateral in the vault
/// @return value The real amount of collateral in the vault
function _collateral(Context memory context) public view returns (Fixed6 value) {
    value = Fixed6Lib.from(UFixed6Lib.from(asset.balanceOf()));
    for (uint256 marketId; marketId < context.markets.length; marketId++)
        value = value.add(context.markets[marketId].collateral);
}
```

https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol/#L486

## Tool used

Manual Review

## Recommendation

Require a bigger initial deposit or premint some shares and burn them before the first deposit. This is a well known attack vector for ERC4626 vaults. You can read more about it [here](https://github.com/transmissions11/solmate/issues/178)