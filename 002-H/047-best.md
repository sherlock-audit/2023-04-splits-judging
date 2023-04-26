ctf_sec

high

# Lack of price oracle output validation can result in wrong or stale price being used in SplitSwapper.sol

## Summary

Stale price and heavily outdated price can be used in Uniswap TWAP oracle

## Vulnerability Detail

Price oracle is important because it the price oracle report wrong price or stale price, the caller of the function flash to get the price at discount while not paying the token beneficiary properly

```solidity
function flash(OracleImpl.QuoteParams[] calldata quoteParams_, bytes calldata callbackData_)
	external
	payable
	pausable
{
	address _tokenToBeneficiary = $tokenToBeneficiary;
	(uint256 amountToBeneficiary, uint256[] memory amountsToBeneficiary) =
		_transferToTrader(_tokenToBeneficiary, quoteParams_);

	ISwapperFlashCallback(msg.sender).swapperFlashCallback({
		tokenToBeneficiary: _tokenToBeneficiary,
		amountToBeneficiary: amountToBeneficiary,
		data: callbackData_
	});

	uint256 excessToBeneficiary = _transferToBeneficiary(_tokenToBeneficiary, amountToBeneficiary);

	emit Flash(msg.sender, quoteParams_, _tokenToBeneficiary, amountsToBeneficiary, excessToBeneficiary);
}
```

this is calling

```solidity
function _transferToTrader(address tokenToBeneficiary_, OracleImpl.QuoteParams[] calldata quoteParams_)
	internal
	returns (uint256 amountToBeneficiary, uint256[] memory amountsToBeneficiary)
{
	amountsToBeneficiary = $oracle.getQuoteAmounts(quoteParams_);
```

and the function call $oracle.getQuoteAmounts use Uniswap V3 Twap oracle

```solidity
function _getQuoteAmount(QuoteParams calldata quoteParams_) internal view returns (uint256) {
	ConvertedQuotePair memory cqp = quoteParams_.quotePair._convert(_convertToken);
	SortedConvertedQuotePair memory scqp = cqp._sort();

	PairOverride memory po = _getPairOverride(scqp);
	if (po.scaledOfferFactor == 0) {
		po.scaledOfferFactor = $defaultScaledOfferFactor;
	}

	// skip oracle if converted tokens are equal
	if (cqp.cBase == cqp.cQuote) {
		return quoteParams_.baseAmount * po.scaledOfferFactor / PERCENTAGE_SCALE;
	}

	if (po.fee == 0) {
		po.fee = $defaultFee;
	}
	if (po.period == 0) {
		po.period = $defaultPeriod;
	}

	address pool = uniswapV3Factory.getPool(scqp.cToken0, scqp.cToken1, po.fee);
	if (pool == address(0)) {
		revert Pool_DoesNotExist();
	}

	// reverts if period is zero or > oldest observation
	(int24 arithmeticMeanTick,) = OracleLibrary.consult({pool: pool, secondsAgo: po.period});

	uint256 unscaledAmountToBeneficiary = OracleLibrary.getQuoteAtTick({
		tick: arithmeticMeanTick,
		baseAmount: quoteParams_.baseAmount,
		baseToken: cqp.cBase,
		quoteToken: cqp.cQuote
	});

	return unscaledAmountToBeneficiary * po.scaledOfferFactor / PERCENTAGE_SCALE;
}
```

the code tries to query the price with arithmeticMeanTic based on po.period, however, the price returned from this query method can be severely outdated and stale.

According to https://docs.uniswap.org/concepts/protocol/oracle

> Historical data is stored as an array of observations. At first, each pool tracks only a single observation, overwriting it as blocks elapse. This limits how far into the past users may access data. However, any party willing to pay the transaction fees may increase the number of tracked observations (up to a maximum of 65535), expanding the period of data availability to ~9 days or more.

this method

https://docs.uniswap.org/contracts/v3/reference/core/UniswapV3Pool#increaseobservationcardinalitynext

```solidity
 function increaseObservationCardinalityNext(
    uint16 observationCardinalityNext
  ) external override lock noDelegateCall
```

basically this mean that the TWAP price of the token from even a week ago can be used.

for example, suppose the base token worth 10 USD quote token a week ago, but now the base token 1000 USD quote token.

it is possible that the oracle still use the 10 USD price from a week ago, which is very outdated, and the caller of flash function use the outdated price to get all a lot base token but pays for low and outdated price in quote token.

## Impact

Stale price and heavily outdated price can be used in Uniswap TWAP oracle and the flash caller in SplitSwapper can get token in wrong price.

## Code Snippet

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-oracle/src/UniV3OracleImpl.sol#L275

## Tool used

Manual Review

## Recommendation

We recommend the validate the price returned from Uniswap V3 oracle to avoid stale price usage.
