

# Architecture
In Uniswap v3, each pool has its own contract instance, which makes making pools and doing swaps in multiple pools costly

![High Level Architecture](/images/01_Pool_Initialization/HighLevelArchitecture.png) 

In v4, all pools are kept in a single contract instance, saving a lot of important gas because tokens don’t have to move between different pool contracts during swaps. Early calculations say that v4 will make the gas cost of creating pools 99% less. Hooks offer unlimited choices and the single contract lets you easily move through all these choices.



![Detailed Architecture](/images/01_Pool_Initialization/DetailedArchitecture.png) 

This Singleton design is improved by a new "flash accounting" method. Instead of moving assets in and out of pools after each swap in v3, this method only moves the net balances. This means the system is a lot more efficient and saves even more gas in Uniswap v4.

Because of the efficiency of the Singleton contract and flash accounting, there is no need to limit fee tiers. People who create pools can choose them to be most competitive or change them with a dynamic fee hook. v4 also supports native ETH again, which helps save more gas.

# PoolManager
To understand the major parts of the PoolManager, let's look at the the main interface it implements: `IPoolManager.sol`.

https://github.com/Uniswap/v4-core/blob/main/contracts/interfaces/IPoolManager.sol 

Here’s a brief overview of the interface.

- It inherits from `IFees` and `IERC1155`, combining fee management and token management functionalities.
- It then defines errors to handle various exceptional conditions like max currencies touched, currency not settled, and improper function caller.
- It then includes the `Initialize`, `ModifyPosition`, and `Swap` events which get emitted during respective operations, helping in tracking state changes.
- It then has the `initialize` function which is used to initialize a new pool.
- Various functions are declared to manage and query pool states like `getSlot0`, `getLiquidity`, and `getPosition`.
- There are functions to manage liquidity positions in pools such as `modifyPosition` and `swap`, which facilitate liquidity provision and token swaps.
- It includes mechanisms to manage pool locks and concurrency through functions like `lock`.
- Functions like `mint`, `take`, and `settle` manage user balances, allowing users to interact with their funds within pools.
- It allows external contracts to access pool state through the `extsload` function, facilitating integration with other contracts or interfaces.
- Custom structs like `ModifyPositionParams` and `SwapParams` are declared to bundle parameters neatly, improving code readability and usability.

# Pool Initialization
The `initialize` function sets up a new liquidity pool in Uniswap. It takes necessary information such as currencies 
and pricing info, and hook information as inputs, checks various conditions to ensure that the pool is set up correctly, 
and sets initial values for the pool. 

After all the checks and setup, it announces that a new pool has been created by triggering an `Initialize` event.

# PoolManager.sol
https://github.com/Uniswap/v4-core/blob/main/contracts/PoolManager.sol

```solidity
/// @inheritdoc IPoolManager
function initialize(PoolKey memory key, uint160 sqrtPriceX96, bytes calldata hookData)
external
override
returns (int24 tick)
{
    if (key.fee.isStaticFeeTooLarge()) revert FeeTooLarge();

    // see TickBitmap.sol for overflow conditions that can arise from tick spacing being too large
    if (key.tickSpacing > MAX_TICK_SPACING) revert TickSpacingTooLarge();
    if (key.tickSpacing < MIN_TICK_SPACING) revert TickSpacingTooSmall();
    if (key.currency0 > key.currency1) revert CurrenciesInitializedOutOfOrder();
    if (!key.hooks.isValidHookAddress(key.fee)) revert Hooks.HookAddressNotValid(address(key.hooks));

    if (key.hooks.shouldCallBeforeInitialize()) {
        if (key.hooks.beforeInitialize(msg.sender, key, sqrtPriceX96, hookData) != IHooks.beforeInitialize.selector)
        {
            revert Hooks.InvalidHookResponse();
        }
    }

    PoolId id = key.toId();
    uint24 protocolFees = _fetchProtocolFees(key);
    uint24 hookFees = _fetchHookFees(key);
    tick = pools[id].initialize(sqrtPriceX96, protocolFees, hookFees);

    if (key.hooks.shouldCallAfterInitialize()) {
        if (
            key.hooks.afterInitialize(msg.sender, key, sqrtPriceX96, tick, hookData)
                != IHooks.afterInitialize.selector
        ) {
            revert Hooks.InvalidHookResponse();
        }
    }

    emit Initialize(id, key.currency0, key.currency1, key.fee, key.tickSpacing, key.hooks);
}
```
Here's what initialize does step-by-step:

1. **Check the Fee**:
   - If the fee for using the pool is too high, then revert error called `FeeTooLarge`.

2. **Check the Tick Spacing**:
   - "Ticks" are like markers in the pool. There's a space between these markers.
   - If the space between the markers is too large, then revert with error `TickSpacingTooLarge`.
   - If it's too small, then revert with error `TickSpacingTooSmall`.

3. **Ensure the Digital Currencies are in Order**:
   - Keys should be sorted numerically
   - If they are in the wrong order, then revert with error `CurrenciesInitializedOutOfOrder`.

4. **Check the Hooks**:

5. **Set Up the Pool**:
   - Create a unique `id` for the pool.
   - Actually set up the pool with the given starting price and fees.

6. **After Setup Actions**:
   - Some hooks might have actions to do after the pool is set up. If they don't respond correctly, revert with `InvalidHookResponse`.

7. **Announce the Pool**:
   - Finally, announce to everyone that a new pool has been created, sharing all its details.

# PoolKey

The `PoolKey` is a structure that uniquely identifies a liquidity pool by storing essential details like the two 
currencies involved (sorted numerically), the swap fee, tick spacing, and hooks (extra functionalities) of the pool. 

It acts as a unique identifier, ensuring that each pool can be precisely specified and accessed within the code.

```solidity
/// @notice Returns the key for identifying a pool
struct PoolKey {
    /// @notice The lower currency of the pool, sorted numerically
    Currency currency0;
    /// @notice The higher currency of the pool, sorted numerically
    Currency currency1;
    /// @notice The pool swap fee, capped at 1_000_000. The upper 4 bits determine if the hook sets any fees.
    uint24 fee;
    /// @notice Ticks that involve positions must be a multiple of tick spacing
    int24 tickSpacing;
    /// @notice The hooks of the pool
    IHooks hooks;
}
```
