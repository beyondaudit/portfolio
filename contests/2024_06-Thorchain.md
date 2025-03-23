# Thorchain

## [H-01] When dealing with native coin, the `TransferOut*` events are still triggered on error

### Impact

The Bifrost might intercept events while nothing actually happened on the blockchain, no state change whatsoever.

Actions might be overtaken by Bifrost once it intercepts such a `TransferOut*` event, which could mess up internal accounting or lead to unexpected behaviors.

### Proof of concept

The file `smartcontract_log_parser.go` is responsible for intercepting events emitted by `THORChain_Router.sol`, notably `TransferOut()` (corresponding to the `transferOutEvent` variable in the `go` file) and `TransferOutAndCall()` (corresponding to the `transferOutAndCallEvent` variable in the `go` file) .

These events are parsed later on in a switch statement which triggers internal mechanisms to take it into account.

There is a scenario in which nothing actually happens on the blockchain but the event is still emitted, which could lead to unexpected behaviors.

Consider the `transferOutAndCall()` function and the following scenario :

<https://github.com/code-423n4/2024-06-thorchain/blob/main/chain/ethereum/contracts/THORChain_Router.sol#L261-L293>

* a low-level call to the `swapOut()` function of the aggregator is made to swap native coin to an ERC20 before sending them to the recipient

```solidity
uint256 _safeAmount = msg.value;
(bool erc20Success, ) = aggregator.call{value: _safeAmount}(
    abi.encodeWithSignature(
        "swapOut(address,address,uint256)",
        finalToken,
        to,
        amountOutMin
    )
);
```

* the previous fails (for some reason) and as a fallback, the native coin is sent directly to the recipient

```solidity
if (!erc20Success) {
    bool ethSuccess = payable(to).send(_safeAmount); // If can't swap, just send the recipient the ETH
    if (!ethSuccess) {
        payable(address(msg.sender)).transfer(_safeAmount); // For failure, bounce back to vault & continue.
    }
}
```

* the native coin transfer also fails (for some other reason) so these coins are sent back to the caller (e.g. the vault)


```solidity
if (!ethSuccess) {
    payable(address(msg.sender)).transfer(_safeAmount); // For failure, bounce back to vault & continue.
}
```

* at this point, the state of the blockchain has not changed, there was only a native coin transfer from the vault to the router contract which sent these native coin back to the vault again

* because the transaction did not revert, the `TransferOutAndCall()` event is still triggered to notify the Bifrost to undertake internal actions.

```solidity
if (!erc20Success) {
    bool ethSuccess = payable(to).send(_safeAmount); // If can't swap, just send the recipient the ETH
    if (!ethSuccess) {
        payable(address(msg.sender)).transfer(_safeAmount); // For failure, bounce back to vault & continue.
    }
}

emit TransferOutAndCall(
    msg.sender,
    aggregator,
    _safeAmount,
    finalToken,
    to,
    amountOutMin,
    memo
);
```

The described issue has been discussed with the team but the consequences of it are still unclear and should undergo an in-depth investigation in my opinion. From my experience, a piece of code that is executed for no particular reason can have terrible effects.

Note this behavior affects the following functions :
* public `transferOut()`
* internal `_transerOutV5()`
* public `transferOutAndCall()`
* internal `_transferOutAndCallV5()`

Due to time constraints, I won't be able to investigate before the contest ends but I'll make sure to give a feedback to the protocol team on Discord and notify the judges during the review.

### Recommended mitigation steps

The transaction should revert instead of sending the native coin to the vault and emitting the corresponding event in order to avoid a mislead on Bifrost.

In case the transaction __MUST NOT EVER__ be reverted, adding a specific value to the event and parsing it differently in the log parser when this happens.

## [M-01] The new `_transferOutAndCallV5()` function is not compatible with fee-on-transfer and rebase tokens

### Impact

1. The function can't ever be executed as intended.

2. Potential loss of user funds

### Proof of concept

The `_transferOutAndCallV5()` function is an internal function that is used in `transferOutAndCallV5()` and `batchTransferOutAndCallV5()` which allows a THOR Vault to transfer an amount of native coin or ERC20 tokens to a recipient by first swapping the corresponding amount using an aggregator.

Here is the code snippet responsible for swapping 1 asset to another before sending the output asset to the recipient :

<https://github.com/code-423n4/2024-06-thorchain/blob/main/chain/ethereum/contracts/THORChain_Router.sol#L342-L375>

```solidity
_vaultAllowance[msg.sender][
    aggregationPayload.fromAsset
] -= aggregationPayload.fromAmount; // Reduce allowance

// send ERC20 to aggregator contract so it can do its thing
(bool transferSuccess, bytes memory data) = aggregationPayload
    .fromAsset
    .call(
        abi.encodeWithSignature(
        "transfer(address,uint256)",
        aggregationPayload.target,
        aggregationPayload.fromAmount
        )
    );

require(
    transferSuccess && (data.length == 0 || abi.decode(data, (bool))),
    "Failed to transfer token before dex agg call"
);

// add test case if aggregator fails, it should not revert the whole transaction (transferOutAndCallV5 call succeeds)
// call swapOutV5 with erc20. if the aggregator fails, the transaction should not revert
(bool _dexAggSuccess, ) = aggregationPayload.target.call{value: 0}(
    abi.encodeWithSignature(
        "swapOutV5(address,uint256,address,address,uint256,bytes,string)",
        aggregationPayload.fromAsset,
        aggregationPayload.fromAmount,
        aggregationPayload.toAsset,
        aggregationPayload.recipient,
        aggregationPayload.amountOutMin,
        aggregationPayload.payload,
        aggregationPayload.originAddress
    )
);
```

First, the `THORChain_Router` contract (that holds the asset to be swapped) transfers the input tokens (`aggregationPayload.fromAsset`) to the aggregator (`aggregationPayload.target`) using a low-level call.

At this point, the aggregator holds the input assets and new low-level call is made to the aggregator to swap these tokens to the output assets using `swapOutV5()`. 

When dealing with fee-on-transfer and rebase tokens, the actual number of tokens received after a `transfer()` might not be equal to the amount of tokens sent.

This means the call to `swapOutV5()` on the aggregator will most likely fail because it will attempt to swap an amount of tokens that exceeds balance of the `THORChain_Router`.

However, the swap won't revert the transaction which will end-up successfully (this is intended as stated by the comment above the call to `swapOutV5()`).

To be clearer, here is an example scenario :

1. Vault executes the `transferOutAndCallV5()` function with these parameters :
* `aggregationPayload.fromAsset` : TKNA which is a fee-on-transfer token
* `aggregationPayload.toAsset` : ETH (native)
* `aggregationPayload.fromAmount` : 1000 (TKNA) 

2. The `transfer()` is executed and sends the 1000 TKNA to the aggregator

3. The aggregator receives 999 TKNA (the fee is 0.1%)

4. Now `swapOutV5()` is called on the aggregator with `aggregationPayload.fromAmount` equal to 1000 still

5. The 1000 amount of TKNA will be passed as a parameter to `swapExactTokensForETH()` on a router (e.g. Uniswap) which will attempt to pull 1000 TKNA from the aggregator resulting in a revert due to an insufficient amount of tokens.

The above scenario can be even more dramatic if the contract holds some TKNA that belong to another user.

In this case, the swap will go through and the 1 missing TKNA levied as fees will be "stolen" from the other user.

The described scenario will now affect that other user.

### Recommended mitigation steps

Cache the balance of the input token the aggregator holds before the transfer then subtract it from the balance the aggregator effectively holds after the transfer and use it as the `fromAmount` in the `swapOutV5()` call.
