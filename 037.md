J4de

medium

# `SwapperImpl.sol#flash` does not set the minimum payment amount, which may be attacked by price manipulation

## Summary

`SwapperImpl.sol#flash` does not set the minimum payment amount, which may be attacked by price manipulation

## Vulnerability Detail

```solidity
File: splits-swapper/src/SwapperImpl.sol
203     function flash(OracleImpl.QuoteParams[] calldata quoteParams_, bytes calldata callbackData_)
204         external
205         payable
206         pausable
207     {
208         address _tokenToBeneficiary = $tokenToBeneficiary;
209         (uint256 amountToBeneficiary, uint256[] memory amountsToBeneficiary) =
210             _transferToTrader(_tokenToBeneficiary, quoteParams_);
211
212         ISwapperFlashCallback(msg.sender).swapperFlashCallback({
213             tokenToBeneficiary: _tokenToBeneficiary,
214             amountToBeneficiary: amountToBeneficiary,
215             data: callbackData_
216         });
217
218         uint256 excessToBeneficiary = _transferToBeneficiary(_tokenToBeneficiary, amountToBeneficiary);
219
220         emit Flash(msg.sender, quoteParams_, _tokenToBeneficiary, amountsToBeneficiary, excessToBeneficiary);
221     }
```

The official document describes splits-swapper as follows:

> uses discount oracle pricing to incentivize third parties to automatically convert multi-token revenue into a single token & forward to beneficiary

The problem here is that the `flash` function does not allow users to set the maximum payment amount, which may cause users to be exposed to price manipulation attacks (such as sandwich attacks) when exchanging tokens.

## Impact

Users may lose part of their funds

## Code Snippet

https://github.com/0xSplits/splits-swapper/blob/83ce1124767a097aac37d1cd162a9b27bbc48701/src/SwapperImpl.sol#L203-L221

## Tool used

Manual Review

## Recommendation

It is recommended to provide the maximum payment amount as an input parameter