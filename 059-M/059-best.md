Fresh Aegean Sparrow

medium

# Vault.sol: Keeper fee accounting is bricked when a user calls update multiple times
Due to the way Vault calculates the share of keeper fees that each account will pay i.e. by dividing globalKeeperFees by Checkpoint.count(number of times Vault#update was called), a single account can game the system by `update`ing many times, to reduce to dust the amount of keeperFees that will be accounted for.

## Vulnerability Detail
Because of the keeper fee that a market deducts when updating a position, Vault.sol accounts for this by charging its users.
Whenever a user updates his position in Vault.sol, during the next settlement, an amount of fee is deducted from each depositor during that epoch. It is expected that the total fees that would be deducted from all users in the epoch equals the global keeperFee for that epoch(which is read from the underlying market(s)).

Vault wrongly calculates the amount of keeperFee that each depositor will pay by dividing globalKeeperFee by `Checkpoint.count`

```solidity
function _withoutKeeperLocal(Checkpoint memory self, UFixed6 amount) private pure returns (UFixed6) {
    UFixed6 keeperPer = self.count == 0 ? UFixed6Lib.ZERO : self.keeper.div(UFixed6Lib.from(self.count));
    return _withoutKeeper(amount, keeperPer);
}
```

`Checkpoint.count` is [incremented](https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Checkpoint.sol#L68) each time a user calls `Vault#update`.

When a user calls `Vault#update` multiple times at a particular checkpoint, the keeper fee that will be debited from that user's account is much less than it should be(depending on the number of times User calls `Vault#update`).

### Example Scenario

- Vault's underlying market is MarketA, which charges a keeper fee of 2DSU
- Alice deposits 10DSU to vault
  - Checkpoint.count is 1, so Alice should pay the full globalKeeperFees of 2DSU during next settlement if she is the only one that updated account in that checkpoint
- Alice calls Vault#update again with 0 deposits
  - globalKeeperFees remains at 2DSU because no new order was created in underlying market
  - Checkpoint.count is now 2
- During the next settlement, since globalKeeperFee is 2DSU, and Alice was the only one that deposited during that epoch, Alice should pay the full 2DSU
- But now, because protocol divides globalKeeperFees by Checkpoint.count, Alice will pay only (2/2)=1DSU
- Vault was charged 2DSU, while only 1DSU was accounted for.
- Note that the second time Alice called Vault#update, it was with 0 deposit.
- Alice can call update many times with 0 deposits to reduce the keeper fees she would pay to dust, while Vault gets charged 2DSU.(Her aim is to make Checkpoint.count as large as possible)

## Impact
The total Keeper fees that would be debited from Vault depositors will be less than the keeperFee that the underlying market charges the vault.
This could lead to loss of funds as shown in Example Scenariio above.

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L271
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Checkpoint.sol#L68
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Checkpoint.sol#L186

## Tool used

Manual Review

## Recommendation
Checkpoint.count should not be the number of times Vault#update was called at that checkpoint. Instead, it should be the number of unique accounts that updated positions at that checkpoint.
This way, no matter how many times a user updates position at a particular checkpoint, he still accounts for the amount he should pay(globalKeeperFee/uniqueAccountsThatUpdatedAtThatCheckpoint)