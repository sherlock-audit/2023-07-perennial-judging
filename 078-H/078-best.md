Fresh Aegean Sparrow

high

# Market.sol: User can deposit, open & close position, and withdraw in a single transaction

Due to the weak `Position.sol::PositionLib#ready` check(which is used in `Market.sol#_settle` function), a user can deposit, open&close position, and withdraw collateral+PnL in a single transaction.
This allows attacker to use a flashloan in order to gain massive PnL from the attack

## Vulnerability Detail
Here is an attack Scenario:
In PythOracle, the latestVersion with a recorded price is at timestamp 90

- At timestamp 100, Alice calls `Market#update` to open a tiny position(either maker or taker) so that a new oracle version is requested.
- Alice gets the `updateData` for that requested oracle version, but does not commit the price yet
- At timestamp 110, Alice opens another tiny position so that another oracle version is requested
- Alice gets the updateData, but does not commit the price

  In a single transaction:

- Alice obtains flashloan
- Alice calls `Market#update` to open a massive maker position with flashloan funds as collateral
- Alice calls `PythOracle#commitRequested` to record the price for the first `updateData` she fetched beforehand. Now, `latestVersion` has been updated with timestamp 100
- Alice calls `Market#update` again to close the position
  - Within while loop in \_settle, latestVersion.timestamp(100) is greater than pendingPosition.timestamp(90), so the opened position gets settled, which allows user to be able to close the position
- Alice calls `PythOracle#commitRequested` again to record the price for the second updateData. Now, latestVersion has been updated again with timestamp 110
- Alice calls `Market#update` again to withdraw flashloan+PnL gained, and pay back flashloan fees(if any)
  - Within while loop in \_settle, latestVersion.timestamp(110) is greater than pendingPosition.timestamp(100), so the closed position gets settled, which allows user to be able to withdraw collateral

One of the main reasons for Perennial's delayed settlement is to prevent the use of flashloans, but with this vulnerability, an attacker can use flashloans in order to gain massive PnL, at the expense of other users' funds.

## Impact
Attacker can use a flashloan to open massive positions and gain unfair PnL at the expense of other users' deposited collateral.
The impact of this is HIGH because:

- Delayed settlements, which is a core feature of Perennial is bypassed
- Attacker loses nothing. Worst case scenario for the attacker is that not enough PnL to pay back flashloan fees is gained from the transaction, in which case, the transaction reverts
- Anyone can perform this attack because `Market#update` and `PythOracle#commitRequested` are not restricted.
- Attack can be replayed multiple times to drain Market

## Code Snippet
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L342
https://github.com/sherlock-audit/2023-07-perennial/blob/main/perennial-v2/packages/perennial/contracts/types/Position.sol#L50

## Tool used

Manual Review

## Recommendation

- Position struct should have a timestampOfUpdate field
- In Market#_update, timestampOfUpdate for the pendingPosition should be set to block.timestamp
- In while statement of Market#_settle, ensure that context.latestVersion>pendingPosition.timestampOfUpdate.
  - You may simply alter PositionLib#ready:
 ```solidity
    function ready(Position memory self, OracleVersion memory latestVersion) internal pure returns (bool) {
@>      return latestVersion.timestamp >= self.timestampOfUpdate;
    }
  ```

