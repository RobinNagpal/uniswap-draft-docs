# Background on Fees
In a liquidity pool, users can provide liquidity by depositing tokens. These positions can be modified 
by either adding or removing liquidity. Fees are also involved in these modifications - collected from 
traders who swap tokens in the pool and distributed to liquidity providers.

![Fees](/images/03_fee_calculation/FeeCalculation.png)

# Swap - Step by Step

The swap happens in a loop until the specified amount has been completely used or the price limit 
has been reached. In each iteration, the code calculates how much of the tokens can be swapped at 
the current price level.

The swap keeps iterating until either the specified amount is fully used or the square root of 
the price hits the defined limit (`sqrtPriceLimitX96`).

# Fees in Uniswap v3
In Uniswap v3, there are two types of fees: Liquidity Provider (LP) Fees and Protocol Fees.

## Liquidity Provider (LP) Fees in v3
Whenever a trade (swap) is executed, the trader pays a fee, which is determined by the Uniswap pool's fee rate. The 
fee is a percentage of the trade amount and is added to the pool's liquidity. This fee compensates liquidity providers 
(LPs) for providing liquidity to the pool. The LP fees are implied within the swap calculation and not explicitly 
shown in the code you provided. The fees for LPs are collected through price movement and accumulated in the 
`feeGrowthGlobal0X128` and `feeGrowthGlobal1X128` variables, which track the global fee growth per unit of 
liquidity in terms of `token0` and `token1` respectively.

## Protocol Fees in v3
These are different from LP fees. The Uniswap protocol may take a portion of the fees for itself, which is separate 
from the fees paid to liquidity providers. The `protocolFees` struct within the contract keeps track of the 
accumulated protocol fees in `token0` and `token1`.

https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol
```solidity
  // accumulated protocol fees in token0/token1 units
    struct ProtocolFees {
        uint128 token0;
        uint128 token1;
    }
    /// @inheritdoc IUniswapV3PoolState
    ProtocolFees public override protocolFees;
```
## Liquidity Provider (LP) Fees vs Protocol Fees
In essence, the key difference between the two types of fees is that:

- **LP Fees** are distributed among the liquidity providers according to their share of the pool and are accumulated in the 
`feeGrowthGlobal` variables.
- **Protocol Fees** are extracted from the LP fees and are meant for the Uniswap protocol's treasury or for other 
- purposes as defined by the governance. They are tracked by the protocolFees variable.

## How Fees are Calculated
A fee is calculated for each swap iteration. It is determined based on the liquidity and the amount being 
swapped in that iteration.

Each swapping loop iteration starts a new swap step. As part of each iteration following properties are being calculated:
- `step.feeAmount`: This is the fee calculated in each swap step, and it's dependent on the liquidity and price movement within that step.
- `state.protocolFee`: This accumulates the total protocol fees during the entire swap.
- `feeGrowthGlobalX128`: This keeps track of the global fee growth, which will be used for liquidity providers to calculate their earned fees.


# Fees in Uniswap v4
V4: The fee structure can be dynamic. This means that the fee rate can potentially change with each swap, depending 
on external logic. This logic is encapsulated by the `IDynamicFeeManager`, which is called if `key.fee.isDynamicFee()` 
is true. There's also a provision for static fees, which behaves similarly to V3 if the dynamic fee feature is not used.

V4 introduces the concept of hook fees, where additional fees can be taken as part of the swap process. This is managed 
by the `hookFees` value and the associated logic that involves external hooks that can be called before and after the 
swap (`key.hooks`). These hooks can perform additional logic and, potentially, change the fees according to 
the swap parameters or other external factors.

The FeeAmounts struct in this Pool library represents the amounts of fees collected from a 
liquidity operation for both the protocol and the pool's hook.

```solidity
    struct FeeAmounts {
        uint256 feeForProtocol0;
        uint256 feeForProtocol1;
        uint256 feeForHook0;
        uint256 feeForHook1;
    }

```
Here is a breakdown of each field:

`feeForProtocol0` and `feeForProtocol1`: These are fees that are directed towards the protocol. This is similar to the fees in Uniswap v3.

`feeForHook0` and `feeForHook1`: These are fees related to a specific operation or integration referred to as 
a "hook" in this code. The fees collected from these hooks may be directed towards the entity that provides the hook 
functionality. Similar to the protocol fees, they are specific to two different assets/tokens in the pool.


If you look at the swapping code in v4, the fees are calculated for each swap step just like in Uniswap v3
https://github.com/Uniswap/v4-core/blob/main/contracts/libraries/Pool.sol#L378

In V4, `Swap` event is emitted at the end of the swap function, which now includes the `totalSwapFee` in its 
parameters. This provides transparency about the fees charged in each swap.

```solidity
    /// @notice Emitted for swaps between currency0 and currency1
    /// @param id The abi encoded hash of the pool key struct for the pool that was modified
    /// @param sender The address that initiated the swap call, and that received the callback
    /// @param amount0 The delta of the currency0 balance of the pool
    /// @param amount1 The delta of the currency1 balance of the pool
    /// @param sqrtPriceX96 The sqrt(price) of the pool after the swap, as a Q64.96
    /// @param liquidity The liquidity of the pool after the swap
    /// @param tick The log base 1.0001 of the price of the pool after the swap
    event Swap(
        PoolId indexed id,
        address indexed sender,
        int128 amount0,
        int128 amount1,
        uint160 sqrtPriceX96,
        uint128 liquidity,
        int24 tick,
        uint24 fee
    );
```

# IHookFeeManager
The `IHookFeeManager` interface is used to manage the fees for the hooks. It's defined as follows:

```solidity
/// @notice The interface for setting a fee on swap or fee on withdraw to the hook
/// @dev This callback is only made if the Fee.HOOK_SWAP_FEE_FLAG or Fee.HOOK_WITHDRAW_FEE_FLAG in set in the pool's key.fee.
interface IHookFeeManager {
    /// @notice Gets the fee a hook can take at swap/withdraw. Upper bits used for swap and lower bits for withdraw.
    /// @param key The pool key
    /// @return The hook fees for swapping (upper bits set) and withdrawing (lower bits set).
    function getHookFees(PoolKey calldata key) external view returns (uint24);
}
```

# Hooks Fee Updates
Hooks fee design is still being finalized. See here for more details - https://github.com/Uniswap/v4-core/issues/318
