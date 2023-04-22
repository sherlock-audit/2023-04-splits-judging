obront

high

# Tokens without UniV3 pairs with `tokenToBeneficiary` can be stolen by an attacker

## Summary

Tokens sent to a Swapper that don't share a UniV3 pool with `tokenToBeneficiary` can be stolen, because an attacker can create such a pool with ultra-low liquidity and maintain it for the length of the TWAP, pricing the tokens at ~0 and swapping to steal them.

## Vulnerability Detail

The oracle uses the TWAP price from Uniswap V3 to determine the price of each asset. 

If a pair is not listed on Uniswap V3, the oracle will not work, so swapping the asset will not be permitted. In this case, the user should be able to withdraw the asset themselves using the `execCalls()` function.

This will be a relatively common occurrence that doesn't require obscure tokens. Many combinations of tokens on Uniswap are able to be traded because of multi-step hops (ie they don't have a pool directly, but they share a poolmate). However, these multi-step hops are not provided by the oracle. A quick review of [Uniswap Pairs List](https://info.uniswap.org/pairs#/) shows that this would impact token pairs as common as MATIC/WBTC or MAKER/FRAX.

In these situations, an attacker could steal all of the tokens in question by performing the following:
- Create a Uniswap V3 pool with the Swapper's `tokenToBeneficiary` and the token in question
- Seed it with an incredibly low amount of liquidity at a ratio that values that token in question at ~0
- Maintain this price for the length of the TWAP, which shouldn't be hard with a new, unused pool and low liquidity (ie if the liquidity amount of lower than the gas needed to perform a swap, arbitrage bots will leave it alone)
- Perform a "swap" from this asset to the `tokenToBeneficiary`, stealing the asset and paying ~0 for it

## Impact

Any tokens sent to a Swapper that do not have a pair with `tokenToBeneficiary` in Uniswap V3 can be stolen.

## Code Snippet

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-oracle/src/UniV3OracleImpl.sol#L274-L282

## Tool used

Manual Review

## Recommendation

`UniV3OracleImpl.sol` should require the pool being used as an oracle to meet certain liquidity thresholds or have existed for a predefined period of time before returning the price to the Swapper.