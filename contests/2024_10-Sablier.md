# Sablier Flow

## [L-01] The use of `decimals()` may not work for all tokens

### Summary

When a stream is created, the contract retrieves the token number of decimals using the `decimals()` function, which is part of the ERC20Metadata standard.

<https://github.com/Cyfrin/2024-10-sablier/blob/main/src/SablierFlow.sol#L579>

```solidity
function _create(
    address sender,
    address recipient,
    UD21x18 ratePerSecond,
    IERC20 token,
    bool transferable
)
    internal
    returns (uint256 streamId)
{
    // Check: the sender is not the zero address.
    if (sender == address(0)) {
        revert Errors.SablierFlow_SenderZeroAddress();
    }

    uint8 tokenDecimals = IERC20Metadata(address(token)).decimals();
```

### Vulnerability Details

The issue is that not all ERC20 tokens provide such an interface meaning the call will not work and will revert.

### Impact

Some tokens may not be compatible with the protocol

### Recommendation

Perform a `try/catch` when retrieving the number of decimals. If the call fails, assume the token has 18 decimals.

The protocol can also allow to manually set the token decimal number as a fallback.
