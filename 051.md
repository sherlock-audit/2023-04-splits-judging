ctf_sec

high

# Wrong logic in SwapperImpl#_transferToBeneficiary

## Summary

Wrong logic in SwapperImpl#_transferToBeneficiary

## Vulnerability Detail

the logic in _transferToBeneficiary is not right

```solidity
function _transferToBeneficiary(address tokenToBeneficiary_, uint256 amountToBeneficiary_)
	internal
	returns (uint256 excessToBeneficiary)
{
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
}
```

if the token is ETH, user is overpaying the amountToBeneficiary_ is _payback > amountToBeneficiary_

and the whole ETH balance is transferred to beneficiary instead of the entitled amountToBeneficiary_

if excess ETH is not refunded to msg.sender

if the token is ERC20 token, the excessive token is transferred to benefiniary instead of msg.sender as well.

Also as we can see the flash function is payable as well

```solidity
function flash(OracleImpl.QuoteParams[] calldata quoteParams_, bytes calldata callbackData_)
	external
	payable
	pausable
{
```

the _transferToBeneficiary failed to consider the msg.value sent with the flash function call and does not accumulate that msg.value into the _payback variable state.

## Impact

Excess ETH and ERC20 is not refunded and the _payback variable does not properly track the msg.value sent along with the flash function call

## Code Snippet

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-swapper/src/SwapperImpl.sol#L203

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-swapper/src/SwapperImpl.sol#L227

## Tool used

Manual Review

## Recommendation

We can change the flash function implementation

```solidity
    function flash(OracleImpl.QuoteParams[] calldata quoteParams_, bytes calldata callbackData_)
        external
        payable
        pausable
    {
	       if(msg.value > 0) { _payback += msg.value} 
```

and change the _transferToBeneficiary implementation to make sure the excess fund is refunded properly

```solidity
function _transferToBeneficiary(address tokenToBeneficiary_, uint256 amountToBeneficiary_)
	internal
	returns (uint256 excessToBeneficiary)
{
	address _beneficiary = $beneficiary;
	if (tokenToBeneficiary_._isETH()) {
		if ($_payback < amountToBeneficiary_) {
			revert InsufficientFunds_FromTrader();
		}
		$_payback -= amountToBeneficiary_;

		// send eth to beneficiary
		uint256 ethBalance = address(this).balance;
		excessToBeneficiary = ethBalance - amountToBeneficiary_;
		_beneficiary.safeTransferETH(amountToBeneficiary_);
		msg.sender.safeTransferETH(excessToBeneficiary);
	} else {
		tokenToBeneficiary_.safeTransferFrom(msg.sender, _beneficiary, amountToBeneficiary_);

		// flush excess tokenToBeneficiary to beneficiary
		excessToBeneficiary = ERC20(tokenToBeneficiary_).balanceOf(address(this));
		if (excessToBeneficiary > 0) {
			tokenToBeneficiary_.safeTransfer(msg.sender, excessToBeneficiary);
		}
	}
}
```

also make sure do not use msg.value in for loop,

also I think handling the edge case for ETH and ERC20 can become tricky. I actually recommend the protocol to just wrap the ETH to WETH so the protocol can implement consistent logic to handle the ERC20 transfer and not need to worry about the usage of msg.value.