# Size Credit

## [H-01] Incorrect Overdue Liquidation Protocol Fees Calculation

### Summary

Borrowers who fail to repay their debt on time can be liquidated due to being `OverDue`. For each liquidation, the protocol charges fees. If the borrower is `UnderWater`, fees are 10% (0.1e18). For `OverDue`, fees are 1% (0.01e18). However, the current implementation miscalculates the `protocolProfitCollateralToken` for `OverDue` liquidations, resulting in a fee of 1.3% instead of the intended maximum of 0.25%.

### Description

In the `OverDue` case, the protocol incorrectly calculates `protocolProfitCollateralToken` in the `executeLiquidate` function of the `Liquidate` library.

<https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L75-L127>

```solidity
function executeLiquidate(State storage state, LiquidateParams calldata params)
    external
    returns (uint256 liquidatorProfitCollateralToken)
{
    DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);
    LoanStatus loanStatus = state.getLoanStatus(params.debtPositionId);
    uint256 collateralRatio = state.collateralRatio(debtPosition.borrower);

    emit Events.Liquidate(params.debtPositionId, params.minimumCollateralProfit, collateralRatio, loanStatus);

    // if the loan is both underwater and overdue, the protocol fee related to underwater liquidations takes precedence
    uint256 collateralProtocolPercent = state.isUserUnderwater(debtPosition.borrower)
        ? state.feeConfig.collateralProtocolPercent
        : state.feeConfig.overdueCollateralProtocolPercent;

    //INFO: ratio (totalCollat/totaleDebt) * DebtFutureValue 
    // #Adapt the collateral amt to this specific debt
    uint256 assignedCollateral = state.getDebtPositionAssignedCollateral(debtPosition);
    // Convert USDC debt in WETH (collateral token)
    uint256 debtInCollateralToken = state.debtTokenAmountToCollateralTokenAmount(debtPosition.futureValue);
    uint256 protocolProfitCollateralToken = 0;

    // profitable liquidation
    if (assignedCollateral > debtInCollateralToken) {
        uint256 liquidatorReward = Math.min(
            assignedCollateral - debtInCollateralToken,
            Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
        );
        liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;

        // split the remaining collateral between the protocol and the borrower, capped by the crLiquidation
        uint256 collateralRemainder = assignedCollateral - liquidatorProfitCollateralToken;

        // cap the collateral remainder to the liquidation collateral ratio
        //   otherwise, the split for non-underwater overdue loans could be too much
        uint256 collateralRemainderCap =
            Math.mulDivDown(debtInCollateralToken, state.riskConfig.crLiquidation, PERCENT);

        collateralRemainder = Math.min(collateralRemainder, collateralRemainderCap);

        protocolProfitCollateralToken = Math.mulDivDown(collateralRemainder, collateralProtocolPercent, PERCENT);
    } else {
        // unprofitable liquidation
        liquidatorProfitCollateralToken = assignedCollateral;
    }

    state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);
    state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);
    state.data.collateralToken.transferFrom(
        debtPosition.borrower, state.feeConfig.feeRecipient, protocolProfitCollateralToken
    );

    debtPosition.liquidityIndexAtRepayment = state.data.borrowAToken.liquidityIndex();
    state.repayDebt(params.debtPositionId, debtPosition.futureValue);
}
```

In the `OverDue` case, the `protocolProfitCollateralToken` should be at most 0.25% of the `debtInCollateralToken`. However, the current implementation calculates it as 1.3%.

### Proof of Concept

Consider a borrower with a collateral value of 4000 USDC and a debt of 1000 USDC. When the debt becomes `OverDue`, it is eligible for liquidation. Following the `executeLiquidate` logic:

<https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L95-L112>

```solidity
if (assignedCollateral > debtInCollateralToken) {
    uint256 liquidatorReward = Math.min(
        assignedCollateral - debtInCollateralToken,
        Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
    );
    liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;

    // split the remaining collateral between the protocol and the borrower, capped by the crLiquidation
    uint256 collateralRemainder = assignedCollateral - liquidatorProfitCollateralToken;

    // cap the collateral remainder to the liquidation collateral ratio
    //   otherwise, the split for non-underwater overdue loans could be too much
    uint256 collateralRemainderCap =
        Math.mulDivDown(debtInCollateralToken, state.riskConfig.crLiquidation, PERCENT);

    collateralRemainder = Math.min(collateralRemainder, collateralRemainderCap);

    protocolProfitCollateralToken = Math.mulDivDown(collateralRemainder, collateralProtocolPercent, PERCENT);
}
```

The calculations are as follows:

1. Liquidator Reward:
    * liquidatorReward = (1000 * 0.05) = 50 USDC

2. Liquidator Profit Collateral Token:
    * liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward = 1000 + 50 = 1050 USDC

3. Collateral Remainder:
    collateralRemainder = assignedCollateral - liquidatorProfitCollateralToken = 4000 - 1050 = 2950 USDC

4. Collateral Remainder Cap:
    * collateralRemainderCap = (debtInCollateralToken * state.riskConfig.crLiquidation) / PERCENT = (1000 * 1.3e18) / 1e18 = 1300 USDC

5. Adjusted Collateral Remainder:
    * collateralRemainder = Math.min(2950, 1300) = 1300 USDC

6. Protocol Profit Collateral Token:
    * protocolProfitCollateralToken = (collateralRemainder * collateralProtocolPercent) / PERCENT = (1300 * 0.01e18) / 1e18 = 13 USDC (1.3% of debtInCollateralToken)

This clearly shows that the `protocolProfitCollateralToken` is miscalculated, being 1.3% instead of the intended 0.25%.

### Impact

The incorrect calculation of `protocolProfitCollateralToken` results in protocol fees that do not comply with the documentation. For `OverDue` liquidations, the maximum fee should be 0.25% of the `debtInCollateralToken`.

### Recommendation

This issue only affects `OverDue` liquidations; `UnderWater` liquidations are not impacted. The calculation for `collateralRemainderCap` should be adjusted as follows:

```diff
if (assignedCollateral > debtInCollateralToken) {
    uint256 liquidatorReward = Math.min(
        assignedCollateral - debtInCollateralToken,
        Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
    );

    liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;

    // split the remaining collateral between the protocol and the borrower, capped by the crLiquidation
    uint256 collateralRemainder = assignedCollateral - liquidatorProfitCollateralToken;

    // cap the collateral remainder to the liquidation collateral ratio
    //   otherwise, the split for non-underwater overdue loans could be too much

-   uint256 collateralRemainderCap = Math.mulDivDown(debtInCollateralToken, state.riskConfig.crLiquidation, PERCENT);
+   uint256 collateralRemainderCap = Math.mulDivDown(debtInCollateralToken, state.riskConfig.crLiquidation, PERCENT) - liquidatorProfitCollateralToken;  

    collateralRemainder = Math.min(collateralRemainder, collateralRemainderCap);

    protocolProfitCollateralToken = Math.mulDivDown(collateralRemainder, collateralProtocolPercent, PERCENT);
}
```

This adjustment ensures the `protocolProfitCollateralToken` is correctly calculated as the difference between 130% of the `debtInCollateralToken` and the `liquidatorProfitCollateralToken`, which is 105% of the `debtInCollateralToken`. This change ensures the protocol fees for `OverDue` liquidations are capped at 0.25% of the `debtInCollateralToken`.

## [H-02] Market Fees Calculation Issue

### Summary

When users buy or sell a credit position, fees are applied based on whether they provide the amount IN or the amount OUT. Depending on which amount is specified, different functions are triggered to calculate these fees and the corresponding amounts.

In some cases, the expected reciprocity between input and output amounts and associated fees is not maintained. For example, when Y amount IN results in X amount OUT with W fees, the reverse scenario where X amount IN does not result in Y amount OUT and W fees introduces discrepancies and inequalities among users.

### Description

The issue arises in the `buyCreditMarket` and `sellCreditMarket` functions within `size.sol`. For the sake of clarity, let's focus on `sellCreditMarket`.

The problem manifests when users attempt to purchase a limit offer using the special ID `RESERVED_ID`. The critical function involved is `executeSellCreditMarket`:

<https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L188-L195>

```solidity
function sellCreditMarket(SellCreditMarketParams memory params) external payable override(ISize) whenNotPaused {
    state.validateSellCreditMarket(params);
    uint256 amount = state.executeSellCreditMarket(params);
    if (params.creditPositionId == RESERVED_ID) {
        state.validateUserIsNotBelowOpeningLimitBorrowCR(msg.sender);
    }
    state.validateVariablePoolHasEnoughLiquidity(amount);
}
```

<https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SellCreditMarket.sol#L127-L204>

```solidity
function executeSellCreditMarket(State storage state, SellCreditMarketParams calldata params)
    external
    returns (uint256 cashAmountOut)
{
    emit Events.SellCreditMarket(
        params.lender, params.creditPositionId, params.tenor, params.amount, params.tenor, params.exactAmountIn
    );

    // slither-disable-next-line uninitialized-local
    CreditPosition memory creditPosition;
    uint256 tenor;
    if (params.creditPositionId == RESERVED_ID) {
        tenor = params.tenor;
    } else {
        DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);
        creditPosition = state.getCreditPosition(params.creditPositionId);

        tenor = debtPosition.dueDate - block.timestamp;
    }

    uint256 ratePerTenor = state.data.users[params.lender].loanOffer.getRatePerTenor(
        VariablePoolBorrowRateParams({
            variablePoolBorrowRate: state.oracle.variablePoolBorrowRate,
            variablePoolBorrowRateUpdatedAt: state.oracle.variablePoolBorrowRateUpdatedAt,
            variablePoolBorrowRateStaleRateInterval: state.oracle.variablePoolBorrowRateStaleRateInterval
        }),
        tenor
    );

    uint256 creditAmountIn;
    uint256 fees;

    if (params.exactAmountIn) {
        creditAmountIn = params.amount;

        (cashAmountOut, fees) = state.getCashAmountOut({
            creditAmountIn: creditAmountIn,
            maxCredit: params.creditPositionId == RESERVED_ID ? creditAmountIn : creditPosition.credit,
            ratePerTenor: ratePerTenor,
            tenor: tenor
        });
    } else {
        cashAmountOut = params.amount;

        (creditAmountIn, fees) = state.getCreditAmountIn({
            cashAmountOut: cashAmountOut,
            maxCashAmountOut: params.creditPositionId == RESERVED_ID
                ? cashAmountOut
                : Math.mulDivDown(creditPosition.credit, PERCENT - state.getSwapFeePercent(tenor), PERCENT + ratePerTenor),
            maxCredit: params.creditPositionId == RESERVED_ID
                ? Math.mulDivUp(cashAmountOut, PERCENT + ratePerTenor, PERCENT - state.getSwapFeePercent(tenor))
                : creditPosition.credit,
            ratePerTenor: ratePerTenor,
            tenor: tenor
        });
    }

    if (params.creditPositionId == RESERVED_ID) {
        // slither-disable-next-line unused-return
        state.createDebtAndCreditPositions({
            lender: msg.sender,
            borrower: msg.sender,
            futureValue: creditAmountIn,
            dueDate: block.timestamp + tenor
        });
    }

    state.createCreditPosition({
        exitCreditPositionId: params.creditPositionId == RESERVED_ID
            ? state.data.nextCreditPositionId - 1
            : params.creditPositionId,
        lender: params.lender,
        credit: creditAmountIn
    });
    
    state.data.borrowAToken.transferFrom(params.lender, msg.sender, cashAmountOut);
    state.data.borrowAToken.transferFrom(params.lender, state.feeConfig.feeRecipient, fees);
}
```

In these functions, the calculation of `creditAmountIn`, `cashAmountOut`, and `fees` differs depending on whether `params.amount` is considered as the input amount (`params.exactAmountIn` == `TRUE`) or the output amount (`params.exactAmountIn` == `FALSE`). This differentiation should maintain consistency in both directions to ensure fairness and equality among users.

```solidity
    uint256 creditAmountIn;
    uint256 fees;

    if (params.exactAmountIn) {
        creditAmountIn = params.amount;

        (cashAmountOut, fees) = state.getCashAmountOut({
            creditAmountIn: creditAmountIn,
            maxCredit: params.creditPositionId == RESERVED_ID ? creditAmountIn : creditPosition.credit,
            ratePerTenor: ratePerTenor,
            tenor: tenor
        });
    } else {
        cashAmountOut = params.amount;

        (creditAmountIn, fees) = state.getCreditAmountIn({
            cashAmountOut: cashAmountOut,
            maxCashAmountOut: params.creditPositionId == RESERVED_ID
                ? cashAmountOut
                : Math.mulDivDown(creditPosition.credit, PERCENT - state.getSwapFeePercent(tenor), PERCENT + ratePerTenor),
            maxCredit: params.creditPositionId == RESERVED_ID
                ? Math.mulDivUp(cashAmountOut, PERCENT + ratePerTenor, PERCENT - state.getSwapFeePercent(tenor))
                : creditPosition.credit,
            ratePerTenor: ratePerTenor,
            tenor: tenor
        });
    }
```

`Assertion`: When a specific amount X is used as `creditAmountIn` resulting in `cashAmountOut` and `fees` with `params.exactAmountIn == TRUE`, the same `creditAmountIn` and `fees` should be obtained when `params.exactAmountIn == FALSE`, and the `cashAmountOut` equal the previous `cashAmountOut`.

### Impact

The discrepancies observed in fees and amounts spanned across different amount ranges, impacting various user types. This inconsistency implies that users may receive or pay different amounts for identical transactions, resulting in unequal treatment among users.

Specifically:
* Fee Discrepancy: Users might be charged varying fees for the same transaction based on the path taken through the functions.
* Amount Variance: The amount received or paid by users could differ unexpectedly, potentially causing financial discrepancies.

### Recommendation

Update the fee calculation logic to ensure consistency across different scenarios, accurately calculating fees and amounts for all transactions.

If this is not feasible, update the documentation accordingly or list this issue under known issues.
