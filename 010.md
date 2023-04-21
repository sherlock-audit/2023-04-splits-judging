obront

high

# SwapperCallbackValidation doesn't do anything, opens up users to having contracts drained

## Summary

The `SwapperCallbackValidation` library that is intended to be used by contracts performing swaps does not provide any protection. As a result, all functions intended to be used only in a callback setting can be called any time by any user. In the provided example of how they expect this library to be used, this would result in the opportunity for all funds to be stolen.

## Vulnerability Detail

The `SwapperCallbackValidation` library is intended to be used by developers to verify that their contracts are only called in a valid, swapper callback scenario. It contains the following function to be implemented:

```solidity
function verifyCallback(SwapperFactory factory_, SwapperImpl swapper_) internal view returns (bool valid) {
    return factory_.isSwapper(swapper_);
}
```

This function simply pings the `SwapperFactory` and confirms that the function call is coming from a verified swapper. If it is, we assume that it is from a legitimate callback.

For an example of how this is used, see the (out of scope) UniV3Swap contract, which serves as a model for developers to build contracts to support Swappers.
```solidity
SwapperImpl swapper = SwapperImpl(msg.sender);
if (!swapperFactory.verifyCallback(swapper)) {
    revert Unauthorized();
}
```
The contract goes on to perform swaps (which can be skipped by passing empty `exactInputParams`), and then sends all its ETH (or ERC20s) to `msg.sender`. Clearly, this validation is very important to protect such a contract from losing funds.

However, if we look deeper, we can see that this validation is not nearly sufficient.

In fact, `SwapperImpl` inherits from `WalletImpl`, which contains the following function:
```solidity
function execCalls(Call[] calldata calls_)
    external
    payable
    onlyOwner
    returns (uint256 blockNumber, bytes[] memory returnData)
{
    blockNumber = block.number;
    uint256 length = calls_.length;
    returnData = new bytes[](length);

    bool success;
    for (uint256 i; i < length;) {
        Call calldata calli = calls_[i];
        (success, returnData[i]) = calli.to.call{value: calli.value}(calli.data);
        require(success, string(returnData[i]));

        unchecked {
            ++i;
        }
    }

    emit ExecCalls(calls_);
}
```
This function allows the owner of the Swapper to perform arbitrary calls on its behalf.

Since the verification only checks that the caller is, in fact, a Swapper, it is possible for any user to create a Swapper and pass arbitrary calldata into this `execCalls()` function, performing any transaction they would like and passing the `verifyCallback()` check.

In the generic case, this makes the `verifyCallback()` function useless, as any calldata that could be called without that function could similarly be called by deploying a Swapper and sending identical calldata through that Swapper.

In the specific case based on the example provided, this would allow a user to deploy a Swapper, call the `swapperFlashCallback()` function directly (not as a callback), and steal all the funds held by the contract.

## Impact

All funds can be stolen from any contracts using the `SwapperCallbackValidation` library, because the `verifyCallback()` function provides no protection.

## Code Snippet

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-swapper/src/peripherals/SwapperCallbackValidation.sol#L11-L19

https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-utils/src/WalletImpl.sol#L43-L65

## Tool used

Manual Review

## Recommendation

I do not believe that Swappers require the ability to execute arbitrary calls, so should not inherit from WalletImpl.

Alternatively, the verification checks performed by contracts accepting callbacks should be more substantial — specifically, they should store the Swapper they are interacting with's address for the duration of the transaction, and only allow callbacks from that specific address.