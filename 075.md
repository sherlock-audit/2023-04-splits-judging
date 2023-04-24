Koolex

medium

# Funds can be completely drained from Swapper

## Summary
Funds can be completely drained in case defaultScaledOfferFactor is zero. **Although this is an admin Input validation, it's still considred as a valid issue since the impact is severe**.
As per Sherlock's criteria
> While most of the admin input issues are invalid, there may be some issues where there could be a valid sanity check. [Example(Valid)](https://github.com/sherlock-audit/2022-10-mycelium-judging-new/issues/164)

[Sherlock's Judging Criteria](https://docs.sherlock.xyz/audits/judging/judging#list-of-issue-categories-that-are-not-considered-valid)

## Vulnerability Detail
`_getQuoteAmount` method is used for getting quote amount for a trade. IF the scaledOfferFactor for pair is not set, it takes the defaultScaledOfferFactor set by Oracle owner (i.e. UniV3Oracle). 
```solidity
	PairOverride memory po = _getPairOverride(scqp);
	if (po.scaledOfferFactor == 0) {
		po.scaledOfferFactor = $defaultScaledOfferFactor;
	}

```
https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-oracle/src/UniV3OracleImpl.sol#L252-L255

If defaultScaledOfferFactor was set zero, the QuoteAmount returned will be zero. Thus, amountsToBeneficiary will be zero. The trader then will pay nothing and can receive all the base token funds from the contract.

```solidity
	amountsToBeneficiary = $oracle.getQuoteAmounts(quoteParams_);
```
https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-swapper/src/SwapperImpl.sol#L231

## Impact
Funds can be completely drained by anyone. i.e. all tokens can be withdrawn from the Swapper. Please not that this impacts all Swappers that uses this Oracle (provided by Splits team).

## Code Snippet

Check above


## Tool used

Manual Review

## Recommendation

Disallow any trade if ScaledOfferFactor is zero to protect the swapper owner.
 
 
  