ctf_sec

high

# Reentrancy in SwapperImpl.sol#flash

## Summary

Reentrancy in SwaperImpl#flash

## Vulnerability Detail

the code is a classical example that violates check effect pattern and trying to update state after external call and vulnerable to reentrancy

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

the external call _transferToTrader(_tokenToBeneficiary, quoteParams_) and ISwapperFlashCallback(msg.sender).swapperFlashCallback

_transferToTrader(_tokenToBeneficiary, quoteParams_) push base token amount to trader

```solidity
	if (amountToTrader > tokenToTrader._balanceOf(address(this))) {
		revert InsufficientFunds_InContract();
	}

	amountToBeneficiary += amountsToBeneficiary[i];
	tokenToTrader._safeTransfer(msg.sender, amountToTrader);
```

note the function call:

```solidity
	tokenToTrader._safeTransfer(msg.sender, amountToTrader);
```

this is calling

```solidity
function _safeTransfer(address token, address addr, uint256 amount) internal {
	if (_isETH(token)) addr.safeTransferETH(amount);
	else token.safeTransfer(addr, amount);
}
```

if the receiver a smart contract and implement payable receive callback if the token is ETH or if the ERC20 token is ERC777 token, the malicious actor can reenter the call flash call, 

ISwapperFlashCallback(msg.sender).swapperFlashCallback can reenter the call as well.

which states is updated after the external call? as we can see the _payback state is updated after externall call

```solidity
address _beneficiary = $beneficiary;
if (tokenToBeneficiary_._isETH()) {
	if ($_payback < amountToBeneficiary_) {
		revert InsufficientFunds_FromTrader();
	}
	$_payback = 0;

	// send eth to beneficiary
	uint256 ethBalance = address(this).balance;
	excessToBeneficiary = ethBalance - amountToBeneficiary_;
	_beneficiary.safeTransferETH(ethBalance);
} else {
	tokenToBeneficiary_.safeTransferFrom(msg.sender, _beneficiary, amountToBeneficiary_);

	// flush excess tokenToBeneficiary to beneficiary
	excessToBeneficiary = ERC20(tokenToBeneficiary_).balanceOf(address(this));
	if (excessToBeneficiary > 0) {
		tokenToBeneficiary_.safeTransfer(_beneficiary, excessToBeneficiary);
	}
}
```

## Impact

Now we can reason about the impact, the reentrancy in SwaperImpl#flash allows user to reenter the call to use the same stale price oracle and stale balance update to sweep base amount fund and also avoid paying the total ETH fee (because _payback is set to 0 after the reentrant call).

## Code Snippet

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-swapper/src/SwapperImpl.sol#L203

## Tool used

Manual Review

## Recommendation

We recommend the protocol use reentrancy guard to protect the flash function call in SwapperImpl
