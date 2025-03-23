# Gamma Brevise Rewarder

## [M-01] Loss of dust tokens in distribution calculation

### Summary

The `createDistribution()` function in the `GammaRewarder` contract is responsible for creating new reward distributions. It calculates the amount of tokens to be distributed per epoch based on the total distribution amount and the number of epochs.

### Root Cause

In the `createDistribution()` function, there is a potential loss of dust tokens when calculating the `amountPerEpoch`. The calculation is performed as follows:

<https://github.com/sherlock-audit/2024-10-gamma-rewarder/blob/main/GammaRewarder/contracts/GammaRewarder.sol#L127>

```solidity
uint256 amountPerEpoch = realAmountToDistribute / ((_endBlockNum - _startBlockNum) / blocksPerEpoch);
```

This division operation may result in a loss of precision due to integer division. Any remainder from this division is effectively lost, leading to a small amount of tokens (dust) being left undistributed.

### Impact

Impact: Low.

The impact of this issue is relatively small, as it only affects a minimal amount of tokens (dust) per distribution. However, over time and with multiple distributions, these small amounts could accumulate to a more significant value.

Likelihood: High.

This issue will occur in almost every distribution where the total amount is not perfectly divisible by the number of epochs. Given the nature of token amounts and block numbers, it's highly likely that most distributions will be affected.

### Proof of Concept

Consider the following scenario:

1. An incentivizor creates a distribution with `realAmountToDistribute` of 1000 tokens.
2. The distribution is set to last for 100 blocks, with `blocksPerEpoch` set to 10.
3. This results in 10 epochs (100 / 10).
4. The `amountPerEpoch` calculation would be: 1000 / (100 / 10) = 1000 / 10 = 100.
5. The total amount distributed over all epochs would be 100 * 10 = 1000.

In this case, there's no loss. However, if we change the `realAmountToDistribute` to 1001:

1. `amountPerEpoch` would still be 1001 / 10 = 100 (integer division).
2. The total amount distributed would be 100 * 10 = 1000.
3. 1 token (dust) remains undistributed and effectively locked in the contract.

### Mitigation

To address this issue, consider using a two-step distribution process:

1. Calculate the `amountPerEpoch` as before.
2. Calculate the total amount that will be distributed using this `amountPerEpoch`.
3. Calculate the remaining dust and add it to the last epoch's distribution.

Here's a code snippet illustrating this approach:

```solidity
uint256 amountPerEpoch = realAmountToDistribute / ((_endBlockNum - _startBlockNum) / blocksPerEpoch);
uint256 totalDistributed = amountPerEpoch * ((_endBlockNum - _startBlockNum) / blocksPerEpoch);
uint256 dust = realAmountToDistribute - totalDistributed;

// Store the amountPerEpoch and dust in the DistributionParameters struct
newDistribution.distributionAmountPerEpoch = amountPerEpoch;
newDistribution.dust = dust;
```

Then, in the claiming process, add the dust to the last epoch's distribution. This ensures that all tokens are distributed, preventing any locked dust in the contract.
