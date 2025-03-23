# Ark Project

## [H-01] Denial of Service attack via unbounded growth of `_collection` array

## Summary

An attacker can exploit the lack of collection removal mechanism to indefinitely increase the `_collection` array size, leading to out-of-gas errors and denial of service.

## Vulnerability Details

1. The attacker initiates multiple withdrawals from L2 to L1 with arbitrary L2 collections when `white_list_enabled` is false.
2. Each withdrawal creates a new collection on L1, increasing the `_collection` array.
3. There's no mechanism to remove collections from `_collection`.
4. The `getWhiteListedCollections` function iterates over the entire `_collection` array.

## Impact

1. The `withdrawTokens` function become unusable due to out-of-gas errors.
2. The `getWhiteListedCollections` function fail, breaking dependent functionalities.
3. Forced whitelisting of collections, compromising the protocol's security model.

## Tools Used

Manual review

## Recommendations

1. Implement a mechanism to remove unused collections from `_collection`.
2. Add a limit to the number of collections that can be added in a given time frame.
3. Implement pagination for `getWhiteListedCollections` to avoid gas limit issues.

## [H-02] Inability to bridge back NFTs due to unupdated L1->L2 mapping

## Summary

When a new NFT collection is bridged from L1 to L2, the L2 contract updates its mapping, but the L1 contract does not. This asymmetry prevents users from bridging their NFTs back to L1, as the verification process fails due to mismatched mappings.

## Vulnerability Details

In `bridge.cairo`, when a new collection is deployed, the mappings are updated:

```cairo
self.l1_to_l2_addresses.write(l1_req, l2_addr_from_deploy);
self.l2_to_l1_addresses.write(l2_addr_from_deploy, l1_req);
```

However, there's no corresponding update on the L1 side.
Then, when attempting to bridge back, the `_verifyRequestAddresses` function in `CollectionManager.sol` will fail:

```solidity
address l1Mapping = _l2ToL1Addresses[collectionL2];
uint256 l2Mapping = snaddress.unwrap(_l1ToL2Addresses[l1Req]);

if (l2Req > 0 && l1Req > address(0)) {
    if (l1Mapping != l1Req) {
        revert InvalidCollectionL1Address();
    } else if (l2Mapping != l2Req) {
        revert InvalidCollectionL2Address();
    } else {
        // All addresses match, we don't need to deploy anything.
        return l1Mapping;
    }
}
```

## Impact

Users cannot bridge back NFTs from newly deployed collections on L2 to L1, because they are locked on the L2 side, leading to loss of assets or inability to use them on the original chain.

## Tools Used

Manual review

## Recommendations

Implement a mechanism to update the L1 contract's mappings when a new collection is deployed on L2:

* This could be done through a message from L2 to L1 after successful deployment.

## [M-01] No assert on `msg.value` for L1 to L2 messaging in `Starklane` contract

## Summary

The `Starklane` contract on L1 does not properly assert the `msg.value` when sending messages to L2. This could lead to messages getting stuck in the bridge due to insufficient fees.

## Vulnerability Details

In the `depositTokens` function, there's no check on the `msg.value` when sending a message to L2:

```solidity
IStarknetMessaging(_starknetCoreAddress).sendMessageToL2{value: msg.value}(
    snaddress.unwrap(_starklaneL2Address),
    felt252.unwrap(_starklaneL2Selector),
    payload
);
```

According to the Cairo Book,  the `msg.value` should be at least 20,000 wei to cover the gas costs of storing the message hash on L1.

## Impact

Without proper assertion of `msg.value`, users might send transactions with insufficient fees, resulting in messages getting stuck in the bridge, then users needing to cancel messages after the 7-day waiting period.

## Tools Used

Manual review

## Recommendations

Add an assertion to check that `msg.value` is within an acceptable range:

```solidity
require(msg.value > 20000 wei, "msg.value should be at least 20,000 wei");

IStarknetMessaging(_starknetCoreAddress).sendMessageToL2{value: msg.value}(
    snaddress.unwrap(_starklaneL2Address),
    felt252.unwrap(_starklaneL2Selector),
    payload
);
```
