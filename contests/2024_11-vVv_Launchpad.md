# vVv Launchpad

## [H-01] MEV Bots Can Front-Run Token Claims Due to Missing Sender Validation

### Summary

The `VVVVCTokenDistributor` contract's `claim()` function lacks proper sender validation, allowing MEV bots to front-run legitimate token claims on Ethereum mainnet (which the protocol is deployed on) by extracting valid signatures from pending transactions in the mempool.

### Root Cause

The [`claim()`](https://github.com/sherlock-audit/2024-11-vvv-exchange-update/blob/main/vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol#L102-L145) function transfers tokens directly to `msg.sender` without validating that the sender matches the KYC address or is an authorized alias:

```solidity
projectToken.safeTransferFrom(
    _params.projectTokenProxyWallets[i],
    msg.sender,
    _params.tokenAmountsToClaim[i]
);
```

### Internal pre-conditions

1. Contract is deployed on Ethereum mainnet
2. Valid signature from authorized signer exists
3. Tokens are available for claiming

### External pre-conditions

1. MEV infrastructure is active and monitoring the mempool
2. User submits a claim transaction with valid parameters

### Attack Path

1. Alice obtains a valid signature to claim tokens
2. Alice submits a claim transaction
3. MEV bot detects the transaction in mempool
4. Bot extracts signature from the transaction parameters
5. Bot front-runs with higher gas price using same parameters but different recipient
6. Original transaction fails as tokens are already claimed

### Impact

* 100% loss of claimed tokens for legitimate users
* Direct financial impact
* Affects all claim transactions with meaningful value

### Mitigation

Add sender validation in the claim function:

```solidity
function claim(ClaimParams memory _params) public {
    // Existing checks...
    
    require(
        msg.sender == _params.kycAddress || 
        isAuthorizedAlias(msg.sender, _params.kycAddress),
        "Sender must be KYC address or authorized alias"
    );
    
    // Rest of function...
}
```

Additionally, implement account abstraction to allow recipient specification in the signed message.
