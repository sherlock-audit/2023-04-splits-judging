obront

medium

# Pairs with liquid UniV2 pools but illiquid UniV3 pools are prone to oracle manipulation

## Summary

## Vulnerability Detail

The Uniswap V3 Oracle works based on the assumption that a Uniswap pool cannot be held in a manipulated state for an ongoing period of time.

However, this assumption is only true if there is an active and sufficiently liquid UniV3 pool. For many token pairs, the more active pairs still exist on Uniswap V2, which means that there is liquidity to trade these tokens, but the UniV3 price is still manipulatable.

As explained in [Euler's Article on Oracle Risk](https://www.euler.finance/blog/euler-protocols-oracle-risk-grading-system): 

> In fact, many well-known tokens have with very liquid markets on Uniswap V2, CEXes but barely any liquidity on Uniswap V3. This creates an easy risk vector: manipulate and drain a lending pool based on Uniswap V3 pricing, and sell the stolen assets on more liquid exchanges.

This risk can occur if either (a) there is too little total liquidity in Uniswap V3 (making it cheap to move) or (b) the liquidity is too concentrated in a narrow range (making it incredibly cheap to move to extremes).

As a response to this, Euler implements "Oracle Ratings" on each pool, to ensure users aren't taken advantage of by these high-risk scenarios.

This is particularly dangerous if it's a common pair that can be traded elsewhere, because, despite the seemingly low liquidity, there is actually sufficient liquidity to swap the stolen tokens for a more desired token after the attack without major slippage or losses.

As a result, an attacker would be able to:
- Find a pair of a token held by the Swapper and the `tokenToBeneficiary` with extremely low liquidity
- Manipulate the price by trading one token for the other
- Maintain this price ratio for the length of the TWAP. This will require an additional trade each block to undo whatever arbitrage bots have performed. This may seem like a lot, but with low liquidity, the dollar amounts will be low.
- After sufficient time has passed (even a fraction of the TWAP can lower the value substantially), perform the swap to take the discounted asset.

## Impact

Token pairs with low liquidity in UniV3 will be susceptible to being manipulated and Swapped at a discount.

## Code Snippet

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-oracle/src/UniV3OracleImpl.sol#L274-L284

## Tool used

Manual Review

## Recommendation

Set a liquidity threshold that is required from the pool in order to allow it as an oracle. This could be hardcoded into the UniV3Oracle, or could be a user submitted parameter.