# Starknet Perpetual

## [H-01] Incorrect signed ratio comparison in liquidation health check lead to user loss and protocol insolvency

### Description

This issue behave in the **liquidate** flow, a similar vulnerability exist for the deleverage flow, but the root cause is different, please do not merge these two reports together.

The `Core::liquidate` function uses `_validate_healthy_or_healthier_position` which improperly handles positions with negative Total Value to Total Risk (TVTR) ratios. This validation fails for deleveragable positions (TVTR < 0) even when risk is reduced, preventing necessary liquidations and allowed liquidation.

The check is done in `assert_healthy_or_healthier` called in `liquidated_position_validations`: 

```rust
pub fn assert_healthy_or_healthier(position_id: PositionId, tvtr: TVTRChange) {

    ...

    let before_ratio = FractionTrait::new(tvtr.before.total_value, tvtr.before.total_risk);
    let after_ratio = FractionTrait::new(tvtr.after.total_value, tvtr.after.total_risk);

@>  assert_with_byte_array(
@>      after_ratio >= before_ratio, position_not_healthy_nor_healthier(:position_id),
@>  );
}
```

### Root Cause  

The health check uses raw ratio comparisons without considering:
1. Negative Total Value (TV) directionality
2. Magnitude changes in Total Risk (TR)

### Proof of Concept:

1. Position enters deleveragable state, after a market movement, with TVTR = -0.5 (TV: -$10M, TR: $20M)
2. Operator attempts liquidation that would reduce TR to $15M (new TVTR = ~ -0.67, -10/15)
3. `assert_healthy_or_healthier_position` compares `after_ratio >= before_ratio` -0.67 >= -0.5 → fails
> here the position is in a better state as the TR is reduced
4. Valid risk-reducing liquidation is blocked
5. Position couldn't be set healthier, risking protocol insolvency if asset prices decline further

### Impact

A chain effect will appear, with all the deleveraged position not being able to be cut. The portocol will be insolvent, pushing user to lose funds.
Note: parallel vulnerability type in deleverage flow prevents alternative risk mitigation, rendering protocol risk management systems ineffective.

### Recommendation

Implement directional-aware ratio comparison:

```rust
fn is_healthier(before: TVTR, after: TVTR) -> bool {
    if before.total_value >= 0 {
        return after.ratio >= before.ratio;
    }
    // For negative TV, allow TR reduction or TV improvement
    (after.total_value > before.total_value) || 
    (after.total_risk < before.total_risk)
}
```

## [M-01] Missing stale price validation in funding tick updates enables protocol insolvency through incorrect funding rate enforcement

### Description

The `AssetsComponent::funding_tick` function processes funding index updates without verifying the underlying asset price freshness. This allows operator manipulation of funding rates using stale prices, bypassing the `max_funding_rate` constraint when prices are outdated.

Stale price data renders `validate_funding_rate` ineffective for constraining operator funding index adjustments, as calculations rely on obsolete market values.

### Proof of Concept:

1. Asset price becomes stale (last update > `max_price_interval`)
2. Operator calls `funding_tick` with arbitrary `new_funding_index`
3. `_process_funding_tick` calculates rate using outdated price from `synthetic_timely_data`
4. `validate_funding_rate` check passes because `time_diff * stale_price` creates artificial headroom
5. Protocol accumulates incorrect funding rates, leading to:
   - Undercollateralized positions remaining open
   - Artificial liquidation of properly collateralized positions
   - Risk of protocol insolvency from skewed funding rates

```rust
// src/core/components/assets/assets.cairo
fn _process_funding_tick(
    ref self: ComponentState<TContractState>,
    time_diff: u64,
    max_funding_rate: u32,
    new_funding_index: FundingIndex,
    synthetic_id: AssetId,
) {
    ...
    let mut synthetic_timely_data = self._get_synthetic_timely_data(:synthetic_id);
    // Uses potentially stale price ↓
    validate_funding_rate(
        :synthetic_id,
        index_diff: index_diff.abs(),
        :max_funding_rate,
        :time_diff,
        synthetic_price: self.get_synthetic_price(:synthetic_id),
    );
    ...
}
```

### Recommendation

**Price freshness check in funding tick:**
```diff
fn _process_funding_tick(
    ref self: ComponentState<TContractState>,
    time_diff: u64,
    max_funding_rate: u32,
    new_funding_index: FundingIndex,
    synthetic_id: AssetId,
) {
+   let price_data = self._get_synthetic_timely_data(:synthetic_id);
+   assert(
+       Time::now().sub(price_data.last_price_update) < self.max_price_interval.read(),
+       SYNTHETIC_EXPIRED_PRICE
+   );
    let mut synthetic_timely_data = self._get_synthetic_timely_data(:synthetic_id);

    ...
```
