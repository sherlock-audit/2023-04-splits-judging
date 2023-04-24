obront

high

# Swapper mechanism cannot incentivize ETH-WETH swaps without risking owner funds

## Summary

## Vulnerability Detail

When `flash()` is called on the Swapper contract, pairs of tokens are passed in consisting of (a) a base token, which is currently held by the contract and (b) a quote token, which is the `$tokenToBeneficiary` that the owner would like to receive.

These pairs are passed to the oracle to get the quoted value of each of them:
```solidity
amountsToBeneficiary = $oracle.getQuoteAmounts(quoteParams_);
```
The `UniV3OracleImpl.sol` contract returns a quote per pair of tokens. However, since Uniswap pools only consist of WETH (not ETH) and are ordered by token address, it performs two conversions first: `_convert()` converts ETH to WETH for both base and quote tokens, and `_sort()` orders the pairs by token address.

```solidity
ConvertedQuotePair memory cqp = quoteParams_.quotePair._convert(_convertToken);
SortedConvertedQuotePair memory scqp = cqp._sort();
```
The oracle goes on to check for pair overrides, and gets the `scaledOfferFactor` for the pair being quoted:
```solidity
PairOverride memory po = _getPairOverride(scqp);
if (po.scaledOfferFactor == 0) {
    po.scaledOfferFactor = $defaultScaledOfferFactor;
}
```
The `scaledOfferFactor` is the discount being offered through the Swapper to perform the swap. The assumption is that this will be set to a moderate amount (approximately 5%) to incentivize bots to perform the swaps, but will be overridden with a value of ~0% for the same tokens, to ensure that bots aren't paid for swaps they don't need to perform.

The problem is that these overrides are set on the `scqp` (sorted, converted tokens), not the actual token addresses. For this reason, ETH and WETH are considered identical in terms of overrides.

Therefore, Swapper owners who want to be paid out in ETH (ie where `$tokenToBeneficiary = ETH`) have two options:

1) They can set the WETH-WETH override to 0%, which successfully stops bots from earning a fee on ETH-ETH trades, but will not provide any incentive for bots to swap WETH in the swapper into ETH. This makes the Swapper useless for WETH.

2) They can keep the WETH-WETH pair at the original ~5%, which will incentivize WETH-ETH swaps, but will also pay 5% to bots for doing nothing when they take ETH out of the contract and return ETH. This makes the Swapper waste user funds.

The same issues exist going in the other direction, when `$tokenToBeneficiary = WETH`.

## Impact

Users who want to be paid out in ETH or WETH will be forced to either (a) have the Swapper not function properly for a key pair or (b) pay bots to perform useless actions.

## Code Snippet

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-oracle/src/UniV3OracleImpl.sol#L248-L260

## Tool used

Manual Review

## Recommendation

The `scaledOfferFactor` (along with its overrides) should be stored on the Swapper, not on the Oracle. 

In order to keep the system modular and logically organized, the Oracle should always return the accurate price for the `scqp`. Then, it is the job of the Swapper to determine what discount is offered for which asset.

This will allow values to be stored in the actual `base` and `quote` assets being used, and not in their converted, sorted counterparts.