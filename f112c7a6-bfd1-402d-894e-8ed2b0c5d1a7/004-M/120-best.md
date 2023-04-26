Tricko

medium

# Swapper's `flash` can be frontrunned to prevent swaps.

## Summary
It is possible for calls to `flash`, especially those swapping the entire balances of tokens, to suffer griefing attacks, by frontrunning them and swapping smalls amounts of token to make the target `flash` call to revert.

## Vulnerability Detail
If some caller calls `flash` (txA) on a Swapper contract, another caller can frontrun this transaction with his own call to `flash`(txB) to reduce that contract token balance below the `QuoteParams.baseAmount` specified in txA, consequently resulting in txA reverting with `InsufficientFunds_InContract()` error. This is most probable when the `flash` call in txA is used to swap the entire balance of some token in the Swapper contract, so txB only needs to swap the minimum amount of tokens possible to force txA to revert. 

Consider the following scenario. Some Swapper contract has `1e18` USDC to be swapped to ETH, Alice is the first caller of `flash` and Bob is the malicious actor.
1. Alice calls `flash` (txA) with `QuoteParams.baseAmount = 1e18` for the USDC-ETH Pair (the remaining params doesn't matter for this example)
2. Bob see Alice's txA in the mempool, then frontruns it with his own `flash` call (txB) with `QuoteParams.baseAmount = 1` for the USDC-ETH Pair.

Bob txB is sucessfull, so Swapper UDSC balance is now `999999999999999999` USDC

3. When Alice's txA in being processed, it will revert because the actual USDC balance is less than the `QuoteParams.baseAmount` specified in her `flash` call (`999999999999999999 < 1e18`)

This can be exploited by bots competing for the same swap, or by malicous actor trying to block a Swapper's functionality at little cost.

## Impact
There is no loss of funds, neither the attacker gains anything in return as the gas costs would be higher than any possible profit from the minimum amount swaps. But a dedicated attacker, willing to consistently spend small amounts of gas could keep blocking all further calls to `flash` on some Swapper contract, therefore possibly affecting the protocol reputation and reducing further adoption by users.

## Code Snippet
https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-swapper/src/SwapperImpl.sol#L203-L221

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-swapper/src/SwapperImpl.sol#L227-L255

## Tool used
Manual Review

## Recommendation
Consider adding aditional logic to `_transferToTrader` so that when `OracleImpl.QuoteParams.baseAmount == type(uint128).max`, it will swap the selected token entire balance, thus allowing callers to use `flash` without specifying exact amount if they desire to swap the entire balance of some token in the Swapper contract and avoiding the frontrunning issue described before. See the SwapperImpl.sol modified code below for part of the fix. splits-oracle's UniV3OracleImpl.sol would also need to be updated so that it handles the `OracleImpl.QuoteParams.baseAmount = type(uint128).max` correctly.

```diff
diff --git a/SwapperImpl.orig.sol b/SwapperImpl.sol
index 23263e3..057d6c3 100644
--- a/SwapperImpl.orig.sol
+++ b/SwapperImpl.sol
@@ -276,38 +276,41 @@ contract SwapperImpl is WalletImpl, PausableImpl {
     function _transferToTrader(address tokenToBeneficiary_, OracleImpl.QuoteParams[] calldata quoteParams_)
         internal
         returns (uint256 amountToBeneficiary, uint256[] memory amountsToBeneficiary)
     {
         amountsToBeneficiary = $oracle.getQuoteAmounts(quoteParams_);
         uint256 length = quoteParams_.length;
         if (amountsToBeneficiary.length != length) revert Invalid_AmountsToBeneficiary();


         uint128 amountToTrader;
         address tokenToTrader;
         for (uint256 i; i < length;) {
             OracleImpl.QuoteParams calldata qp = quoteParams_[i];


             if (tokenToBeneficiary_ != qp.quotePair.quote) revert Invalid_QuoteToken();
             tokenToTrader = qp.quotePair.base;
             amountToTrader = qp.baseAmount;

+            if (amountToTrader == type(uint128).max) {
+                amountToTrader = tokenToTrader._balanceOf(address(this));
+            }

             if (amountToTrader > tokenToTrader._balanceOf(address(this))) {
                 revert InsufficientFunds_InContract();
             }


             amountToBeneficiary += amountsToBeneficiary[i];
             tokenToTrader._safeTransfer(msg.sender, amountToTrader);


             unchecked {
                 ++i;
             }
         }
     }


     function _transferToBeneficiary(address tokenToBeneficiary_, uint256 amountToBeneficiary_)
         internal
```