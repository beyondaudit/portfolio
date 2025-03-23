# Symbiotic

## [M-01] Misbehaving operator could be spared from slashing

### Description

The issue described affects both the `Slasher` and the `VetoSlasher` contracts but they differ in the way it appears.

The report will focus on the `Slasher` contract in the first place before specifying the flow involved in `VetoSlasher`.

#### Slasher

The network has the ability to slash a misbehaving operator using `Slasher::slash()` given an `amount` to slash as a parameter.

The `Slasher::slash()` function adjusts the slashed amount by taking the lesser value from `BaseSlasher::slashableStake()` and the `amount` parameter.

In case the adjusted amount equals 0, the slashing reverts.

<https://cantina.xyz/code/8bab566e-a6d4-4c1b-9f28-71a94bfd1da2/src/contracts/slasher/Slasher.sol#L43-L47>

```solidity
function slash(
// snip...
    if (
        captureTimestamp < Time.timestamp() - IVault(vault_).epochDuration() || captureTimestamp >= Time.timestamp()
    ) {
        revert InvalidCaptureTimestamp();
    }

    slashedAmount =
@>        Math.min(amount, slashableStake(subnetwork, operator, captureTimestamp, slashHints.slashableStakeHints));
    if (slashedAmount == 0) {
@>        revert InsufficientSlash();
    }
// snip...
```

One way for `slashedAmount` to return 0 in this scenario relies in `BaseSlasher::slashableStake()` if `captureTimestamp < latestSlashedCaptureTimestamp[subnetwork]` : in other words, if a slash has already been captured more recently than the one being executed at the time.

<https://cantina.xyz/code/8bab566e-a6d4-4c1b-9f28-71a94bfd1da2/src/contracts/slasher/BaseSlasher.sol#L97-L102>

```solidity
function slashableStake(
// snip...
    if (
        captureTimestamp < Time.timestamp() - IVault(vault).epochDuration() || captureTimestamp >= Time.timestamp()
@>          || captureTimestamp < latestSlashedCaptureTimestamp[subnetwork]
    ) {
        return 0;
    }
// snip...
```

The function `BaseSlasher::slashableStake()` can also return 0 in case the operator has `not opted-in` for the network at the given `captureTimestamp`.

This can happen because `BaseSlasher::slashableStake()` will call the `BaseDelegator::stakeAt()` function which returns 0 when the given `operator`, at `captureTimestamp` is `NOT opted-in`.

<https://cantina.xyz/code/8bab566e-a6d4-4c1b-9f28-71a94bfd1da2/src/contracts/slasher/BaseSlasher.sol#L97-L102>

```solidity
function slashableStake(
// snip...
@>  uint256 stakeAmount = IBaseDelegator(IVault(vault).delegator()).stakeAt(
        subnetwork, operator, captureTimestamp, slashableStakeHints.stakeHints
    );
@>  return stakeAmount
        - Math.min(
            cumulativeSlash(subnetwork, operator)
                - cumulativeSlashAt(subnetwork, operator, captureTimestamp, slashableStakeHints.cumulativeSlashFromHint),
            stakeAmount
        );
// snip...
```

<https://cantina.xyz/code/8bab566e-a6d4-4c1b-9f28-71a94bfd1da2/src/contracts/delegator/BaseDelegator.sol#L114-L118>

```solidity
function stakeAt(
// snip...    
    if (
        stake_ == 0
@>          || !IOptInService(OPERATOR_VAULT_OPT_IN_SERVICE).isOptedInAt(
                operator, vault, timestamp, stakeBaseHints.operatorVaultOptInHint
            )
@>          || !IOptInService(OPERATOR_NETWORK_OPT_IN_SERVICE).isOptedInAt(
                operator, subnetwork.network(), timestamp, stakeBaseHints.operatorNetworkOptInHint
            )
    ) {
        return 0;
    }
// snip...    
```

In case 2 different operators misbehave and the network attempts to slash both of them at a different `captureTimestamp`, there is a chance for one of them to be spared from slashing : e.g. if the first is slashed at a `captureTimestamp` greater than the second.

This can be an issue for multiple reasons :
* the spared operator could be more misbehaving => it should be slashed a greater amount than the other
* withdrawals on the operator could occur after being spared => the stake at risk is reduced
* after intentionally misbehaving, the operator could opt out => future slashing attempts will also revert

Let the following scenario :
* `alice` and `bob` are misbehaving operators
* `network` detects their activity at 2 different timestamps :
    - `alice` : 500
    - `bob` : 400
* network attempt to slash them with 2 different amounts :
    - `alice` : 1 ether
    - `bob` : 2 ether
* the transactions are broadcasted and executed in this order :
    - `slash(subnetwork, alice, 1 ether)`
    - `slash(subnetwork, bob,   2 ether)`
* the tx that slashes `alice` updates `latestSlashedCaptureTimestamp[subnetwork]` and sets it to `500`
* the tx that slashes `bob` adjusts the slashed amount using `BaseSlasher::slashableStake()` but since the `captureTimestamp` (e.g. 400) is less than `latestSlashedCaptureTimestamp[subnetwork]` (e.g. 500), the function will return 0 which will make the tx revert later.
* however the network attempts to slash bob with a more recent timestamp
* but `bob` has opted-out from the network and can't be slashed

#### VetoSlasher

The `VetoSlasher` contract is also subjected to the issue but instead of reverting due to `BaseSlasher::slashableStake()` returning 0, it reverts directly if `captureTimestamp < latestSlashedCaptureTimestamp[subnetwork]` during `VetoSlasher::executeSlash()` (which can occur in the same circumstances as described above).

<https://cantina.xyz/code/8bab566e-a6d4-4c1b-9f28-71a94bfd1da2/src/contracts/slasher/VetoSlasher.sol#L157>

```solidity
function executeSlash(
// snip...
    address vault_ = vault;
    if (Time.timestamp() - request.captureTimestamp > IVault(vault_).epochDuration()) {
        revert SlashPeriodEnded();
    }

@>  _checkLatestSlashedCaptureTimestamp(request.subnetwork, request.captureTimestamp);
// snip...
```

<https://cantina.xyz/code/8bab566e-a6d4-4c1b-9f28-71a94bfd1da2/src/contracts/slasher/BaseSlasher.sol#L123-L127>

```solidity
function _checkLatestSlashedCaptureTimestamp(bytes32 subnetwork, uint48 captureTimestamp) internal view {
    if (captureTimestamp < latestSlashedCaptureTimestamp[subnetwork]) {
        revert OutdatedCaptureTimestamp();
    }
}
```

### Proof of Concept

Here is a coded PoC that described the previous scenario.

It can be pasted in `test\slasher\Slasher.t.sol` and executed with :

```bash
forge test --match-test test_ConcurrentSlashRevert
```

```solidity
function test_ConcurrentSlashRevert() public {
    uint48 epochDuration = 5 days;
    uint256 depositAmount = 100 ether;
    uint256 slashAmount1 = 1 ether;

    uint256 blockTimestamp = block.timestamp * block.timestamp / block.timestamp * block.timestamp / block.timestamp;
    blockTimestamp = blockTimestamp + 1_720_700_948;
    vm.warp(blockTimestamp);

    (vault, delegator, slasher) = _getVaultAndDelegatorAndSlasher(epochDuration);

    address network = alice;
    _registerNetwork(network, alice);
    _setMaxNetworkLimit(network, 0, type(uint256).max);

    _registerOperator(alice);
    _registerOperator(bob);

    _optInOperatorVault(alice);
    _optInOperatorVault(bob);

    _optInOperatorNetwork(alice, address(network));
    _optInOperatorNetwork(bob, address(network));

    _deposit(alice, depositAmount);

    _setNetworkLimit(alice, network, type(uint256).max);

    _setOperatorNetworkLimit(alice, network, alice, type(uint256).max / 2);
    _setOperatorNetworkLimit(alice, network, bob, type(uint256).max / 2);

    // @audit operator bob intentionally misbehaves and opts out
    blockTimestamp = blockTimestamp + 2;
    vm.warp(blockTimestamp);
    _optOutOperatorNetwork(bob, network);

    // @audit time passes and both alice and bob misbehaviors are detected
    blockTimestamp = blockTimestamp + 8;
    vm.warp(blockTimestamp);

    // @audit operator alice gets slashed 1 ether at `blockTimestamp - 1`
    uint256 aliceSlash = _slash(alice, network, alice, slashAmount1, uint48(blockTimestamp - 1), "");
    
    // @audit operator bob gets slashed 2 ether at `blockTimestamp - 2` (before alice)
    // but since alice's slash occurred first, bob's can't happen
    vm.expectRevert(ISlasher.InsufficientSlash.selector);
    _slash(alice, network, bob, slashAmount1 * 2, uint48(blockTimestamp - 2), "");

    assertEq(
        aliceSlash,
        slashAmount1
    );

    // @audit attempts to slash bob also revert because he opted out early enough
    vm.warp(blockTimestamp + epochDuration);
    for(uint256 i = 10 ; i < 3600 ; i = i + 10) {
        vm.expectRevert(ISlasher.InsufficientSlash.selector);
        _slash(alice, network, bob, slashAmount1 * 2, uint48(blockTimestamp + i), "");
    }
}
```

### Recommendation

To address the issue, 2 aspects must be taken into account :
1. Introduce a variable responsible for accounting the latest slashed capture timestamp of a particular operator and use it in `BaseSlasher::slashableStake()` to determine the adjusted slashable amount.
2. Make the `OptInService::optOut()` function a 2-steps process with a cooldown period.

