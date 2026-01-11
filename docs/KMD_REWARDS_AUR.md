# KMD Rewards (AUR) - Active User Rewards

## Overview

KMD (Komodo) implements a unique reward system called Active User Rewards (AUR) that allows users to earn interest on their KMD holdings. This document describes the algorithm, activation conditions, and implementation details.

## Algorithm Location

The main calculation function is located in:
- **File**: `mm2src/coins/utxo.rs`
- **Function**: `kmd_interest()` (lines 1649-1715)
- **Reference**: Based on [Komodo's komodo_interest.h](https://github.com/KomodoPlatform/komodo/blob/master/src/komodo_interest.h)

## Activation

### When Rewards Are Calculated

Rewards are automatically calculated during transaction generation through the `calc_interest_if_required()` function:

- **File**: `mm2src/coins/utxo/utxo_common.rs`
- **Function**: `calc_interest_if_required()` (lines 853-885)
- **Called from**: Transaction building process (line 768 in `utxo_common.rs`)

The function is invoked when:
1. A transaction is being built
2. The coin supports interest (checked via `supports_interest()`)
3. Input UTXOs are being processed

### Support Check

The system checks if a coin supports interest via the `supports_interest()` method, which is implemented for various UTXO coin types:

- `UtxoStandard` (line 142 in `utxo_standard.rs`)
- `QtumCoin` (line 357 in `qtum.rs`)
- `BchCoin` (line 742 in `bch.rs`)
- `ZCoin` (line 1899 in `z_coin.rs`)
- `Qrc20Coin` (line 676 in `qrc20.rs`)

All implementations check if the coin ticker is "KMD" using the `is_kmd()` helper function:

```rust
pub fn is_kmd<T: UtxoCommonOps>(coin: &T) -> bool {
    &coin.as_ref().conf.ticker == "KMD"
}
```

## Supported Coins

**Only KMD (Komodo) supports rewards.** All `supports_interest()` implementations verify that the coin ticker equals "KMD".

## Conditions for Rewards Accrual

For rewards to be calculated and accrued, the following conditions must be met:

### 1. Minimum Amount
- **Requirement**: UTXO value must be at least **10 KMD**
- **In satoshis**: `value >= 1,000,000,000` (1 billion satoshis)
- **Error**: `UtxoAmountLessThanTen` if not met

### 2. Locktime Requirements
- **Requirement**: Locktime must be set and valid
- **Minimum locktime**: `lock_time >= 500,000,000` (LOCKTIME_THRESHOLD)
- **Errors**: 
  - `LocktimeNotSet` if `lock_time == 0`
  - `LocktimeLessThanThreshold` if `lock_time < 500,000,000`

### 3. Transaction Status
- **Requirement**: Transaction must be confirmed (mined)
- **Error**: `TransactionInMempool` if `height == None`

### 4. Block Height Limit
- **Requirement**: Transaction height must be less than the end-of-era block
- **Limit**: `height < 7,777,777` (KOMODO_ENDOFERA)
- **Error**: `UtxoHeightGreaterThanEndOfEra` if exceeded

### 5. Time Requirements
- **Requirement**: At least 1 hour must pass since the transaction's locktime
- **Calculation**: `(current_time - lock_time) / 60 >= 60` minutes
- **Additional**: `current_time > lock_time`
- **Error**: `OneHourNotPassedYet` if not met

## Calculation Algorithm

### Constants

```rust
const KOMODO_ENDOFERA: u64 = 7_777_777;
const LOCKTIME_THRESHOLD: u64 = 500_000_000;
const N_S7_HARDFORK_HEIGHT: u64 = 3_484_958;  // dPoW Season 7, Fri Jun 30 2023
const MINUTES_PER_YEAR: u64 = 525_600;        // 365 * 24 * 60
const MINUTES_PER_AUR: u64 = 20 * MINUTES_PER_YEAR;  // 20 years in minutes
```

### Accrual Period Limits

The maximum accrual period depends on the block height:

- **Before block 1,000,000**: Rewards accrue for up to **1 year** (525,600 minutes)
- **After block 1,000,000**: Rewards accrue for up to **1 month** (44,640 minutes = 31 * 24 * 60)

### Calculation Formula

The calculation adjusts for the time elapsed and applies different rates based on the hardfork:

1. Calculate elapsed minutes: `minutes = (current_time - lock_time) / 60`
2. Apply accrual period limits (see above)
3. Subtract 59 minutes: `minutes -= 59`
4. Calculate accrued rewards:

**Before Season 7 Hardfork (height < 3,484,958)**:
```rust
accrued = (value / MINUTES_PER_AUR) * minutes
```
- AUR rate: **5% per 20 years**

**After Season 7 Hardfork (height >= 3,484,958)**:
```rust
accrued = (value / MINUTES_PER_AUR) * minutes / 500
```
- AUR rate: **0.01% per 20 years** (reduced by factor of 500)

### KIP-0001 Reduction

The reduction from 5% to 0.01% was proposed in [KIP-0001](https://github.com/KomodoPlatform/kips/blob/main/kip-0001.mediawiki) and implemented in [Komodo PR #584](https://github.com/KomodoPlatform/komodo/pull/584).

## Accrual Timing

### Start Time
Rewards start accruing **1 hour after the transaction's locktime**:
```rust
accrue_start_at = lock_time + 3600  // 1 hour in seconds
```

### Stop Time
Rewards stop accruing after:
- **Before block 1,000,000**: `lock_time + (365 * 24 * 60 * 60)` seconds (1 year)
- **After block 1,000,000**: `lock_time + (31 * 24 * 60 * 60)` seconds (1 month)

## Implementation Details

### Transaction Processing

When building a transaction, the system:

1. Checks if the coin supports interest via `supports_interest()`
2. Sets the transaction locktime to the current median time past (MTP)
3. Iterates through all input UTXOs
4. For each input, calculates interest using `kmd_interest()`
5. Sums all interest values
6. If interest is zero, sets a minimal locktime to allow future claiming

### Reward Information API

The system provides a `kmd_rewards_info()` function that:
- Returns rewards information for all unspent outputs
- Only works for KMD coin
- Orders outputs by value (highest to lowest)
- Includes accrual start/stop times and accrued amounts

**Location**: `mm2src/coins/utxo.rs` (lines 1773-1827)

### Reward Updates

The `update_kmd_rewards()` function recalculates rewards for existing transactions:

- **File**: `mm2src/coins/utxo/utxo_common.rs` (lines 4216-4255)
- Updates transaction fee details to include rewards
- Formula: `actual_fee = TransactionDetails::fee + kmd_rewards`

## Error Reasons

The system defines specific reasons why rewards may not accrue:

```rust
enum KmdRewardsNotAccruedReason {
    LocktimeNotSet,                    // lock_time == 0
    LocktimeLessThanThreshold,          // lock_time < 500,000,000
    UtxoHeightGreaterThanEndOfEra,      // height >= 7,777,777
    UtxoAmountLessThanTen,              // value < 10 KMD
    OneHourNotPassedYet,                // Less than 1 hour elapsed
    TransactionInMempool,               // Transaction not yet mined
}
```

## Examples

### Example 1: Basic Calculation (Pre-S7)
- Value: 100 KMD (10,000,000,000 satoshis)
- Locktime: 1,600,000,000 (Unix timestamp)
- Current time: 1,610,000,000 (10,000,000 seconds later ≈ 115.7 days)
- Height: 2,000,000 (before S7 hardfork)
- Minutes: 166,666 (after subtracting 59)
- Calculation: `(10,000,000,000 / 10,512,000) * 166,666 ≈ 158,730` satoshis ≈ 0.001587 KMD

### Example 2: Post-S7 Calculation
- Same parameters as above
- Height: 4,000,000 (after S7 hardfork)
- Calculation: `(10,000,000,000 / 10,512,000) * 166,666 / 500 ≈ 317` satoshis ≈ 0.00000317 KMD

## Related Functions

- `kmd_interest()` - Main calculation function
- `calc_interest_if_required()` - Called during transaction building
- `calc_interest_of_tx()` - Calculates rewards for an existing transaction
- `update_kmd_rewards()` - Updates rewards for transaction details
- `kmd_rewards_info()` - API function to get rewards information
- `kmd_interest_accrue_start_at()` - Calculates when accrual starts
- `kmd_interest_accrue_stop_at()` - Calculates when accrual stops

## References

- [Komodo Interest Implementation](https://github.com/KomodoPlatform/komodo/blob/master/src/komodo_interest.h)
- [KIP-0001: AUR Reduction](https://github.com/KomodoPlatform/kips/blob/main/kip-0001.mediawiki)
- [Komodo PR #584](https://github.com/KomodoPlatform/komodo/pull/584)

