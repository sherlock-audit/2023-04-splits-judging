0xmuxyz

medium

# Lack of a validation to check whether or not the total percent allocations in the `sortedPercentAllocations` array would be `100%`, which lead to that the percent allocations would be wrongly set

## Summary
Lack of a validation to check whether or not the total percent allocations in the `sortedPercentAllocations` array would be `100%`, which lead to that the percent allocations would be wrongly set. 
For example, the total percent allocations would be more than 100% or less than 100%.


## Vulnerability Detail

Within the DiversifierFactory, the `RecipientParams` struct, the `percentAllocation` would be defined like this:
https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-diversifier/src/DiversifierFactory.sol#L40
```solidity
    struct RecipientParams {
        address account;
        CreateSwapperParams createSwapperParams;
        uint32 percentAllocation; /// @audit
    }
```

Within the DiversifierFactory#`createDiversifier()`, the `sortedPercentAllocations` would be returned from the DiversifierFactory#`_parseRecipientParams()` and then it would be assigned into the `percentAllocations` property like this:
https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-diversifier/src/DiversifierFactory.sol#L77-L78
```solidity
    function createDiversifier(CreateDiversifierParams calldata params_) external returns (address diversifier) {
        // create pass-through wallet w {this} as owner & no passThrough
        PassThroughWalletImpl passThroughWallet = passThroughWalletFactory.createPassThroughWallet(
            PassThroughWalletImpl.InitParams({owner: address(this), paused: params_.paused, passThrough: ADDRESS_ZERO})
        );
        diversifier = address(passThroughWallet);

        // parse oracle params for swapper-recipients
        OracleImpl oracle = _parseOracleParams(diversifier, params_.oracleParams);

        // create split w diversifier (pass-through wallet) as controller
        (address[] memory sortedAccounts, uint32[] memory sortedPercentAllocations) =  /// @audit
            _parseRecipientParams(diversifier, oracle, params_.recipientParams);
        address passThroughSplit = splitMain.createSplit({
            accounts: sortedAccounts,
            percentAllocations: sortedPercentAllocations,    /// @audit
            distributorFee: 0,
            controller: diversifier
        });
```

Within the DiversifierFactory#`_parseRecipientParams()` that is called above, the LibRecipients#`_pack()` would be used for calculating the percent allocation of each recipient and then the info including the calculated-percent allocations would be returned via the LibRecipients#`_unpackAccountsInPlace()` like this:
https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-diversifier/src/DiversifierFactory.sol#L124
```solidity
    function _parseRecipientParams(
        address diversifier_,
        OracleImpl oracle_,
        RecipientParams[] calldata recipientParams_
    ) internal returns (address[] memory, uint32[] memory) {
        ...
        // parse recipient params
        uint256 length = recipientParams_.length;
        PackedRecipient[] memory packedRecipients = new PackedRecipient[](length);
        for (uint256 i; i < length;) {
            RecipientParams calldata recipientParams = recipientParams_[i];
            ...
            packedRecipients[i] = account._pack(recipientParams.percentAllocation);  /// @audit
            ...
        }

        ...
        return packedRecipients._unpackAccountsInPlace();  /// @audit
    }
```

Within the LibRecipients#`_pack()`, the `percentAllocation_` would be wrapped like this:
https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-utils/src/LibRecipients.sol#L129-L131
```solidity
    function _pack(address account_, uint32 percentAllocation_) internal pure returns (PackedRecipient) {
        return PackedRecipient.wrap((uint256(uint160(account_)) << UINT96_BITS) | percentAllocation_);
    }
```

Within the LibRecipients#`_unpackAccountsInPlace()`, the LibRecipients#`_unpack()` would be called and then the `percentAllocations` array would be returned like this:
https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-utils/src/LibRecipients.sol#L88-L99
```solidity
    function _unpackAccountsInPlace(PackedRecipient[] memory packedRecipients_)
        internal
        pure
        returns (address[] memory accounts, uint32[] memory percentAllocations)
    {
        uint256 length = packedRecipients_.length;
        assembly ("memory-safe") {
            accounts := packedRecipients_
        }
        percentAllocations = new uint32[](length);
        for (uint256 i; i < length;) {
            (accounts[i], percentAllocations[i]) = _unpack(packedRecipients_[i]);  /// @audit 
            ...
        }
    }
```

Within the LibRecipients#`_unpack()`, the `percentAllocation` would be unwrapped and returned like this:
https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-utils/src/LibRecipients.sol#L139
```solidity
    function _unpack(PackedRecipient packedRecipient_)
        internal
        pure
        returns (address account, uint32 percentAllocation)
    {
        uint256 packedRecipient = PackedRecipient.unwrap(packedRecipient_);
        percentAllocation = uint32(packedRecipient);  /// @audit 
        account = address(uint160(packedRecipient >> UINT96_BITS));
    }
```

On the assumption that, total percent allocations are supposed to be 100%.

However, through the process of the percent allocation from the DiversifierFactory#`createDiversifier()` to the LibRecipients#`_unpack()` above, there is no validation to check whether or not the total percentage allocations would be 100%. 


## Impact
This lead to that the percent allocations would be wrongly set. 
For example, the total percent allocations would be more than 100% or less than 100%.

## Code Snippet
- https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-diversifier/src/DiversifierFactory.sol#L40
- https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-diversifier/src/DiversifierFactory.sol#L77-L78
- https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-diversifier/src/DiversifierFactory.sol#L124
- https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-utils/src/LibRecipients.sol#L129-L131
- https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-utils/src/LibRecipients.sol#L88-L99
- https://github.com/sherlock-audit/2023-04-splits/blob/main/splits-utils/src/LibRecipients.sol#L139


## Tool used
Manual Review

## Recommendation
Within the DiversifierFactory#`createDiversifier()`, consider adding a validation to check whether or not the total percent allocations in the `sortedPercentAllocations` array would be `100%` like this:
(NOTE：In the example code below, the percent would be assumed in Basis Point)
```solidity
    function createDiversifier(CreateDiversifierParams calldata params_) external returns (address diversifier) {
        // create pass-through wallet w {this} as owner & no passThrough
        PassThroughWalletImpl passThroughWallet = passThroughWalletFactory.createPassThroughWallet(
            PassThroughWalletImpl.InitParams({owner: address(this), paused: params_.paused, passThrough: ADDRESS_ZERO})
        );
        diversifier = address(passThroughWallet);

        // parse oracle params for swapper-recipients
        OracleImpl oracle = _parseOracleParams(diversifier, params_.oracleParams);

        // create split w diversifier (pass-through wallet) as controller
        (address[] memory sortedAccounts, uint32[] memory sortedPercentAllocations) = 
            _parseRecipientParams(diversifier, oracle, params_.recipientParams);

+       uint32 totalPercentAllocations;
+       for (uint i=0; i < sortedPercentAllocations.length) {
+           totalPercentAllocations += sortedPercentAllocations[i];
+       }
+       require(totalPercentAllocations == 100_00, "Total percent allocations must be equal to 100%")  /// @dev - NOTE: The percent would be assumed in Basis Point (BPS)

        address passThroughSplit = splitMain.createSplit({
            accounts: sortedAccounts,
            percentAllocations: sortedPercentAllocations,
            distributorFee: 0,
            controller: diversifier
        });
```
