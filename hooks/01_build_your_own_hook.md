# Build your own hook
Whenever starting on a new project, it is always a good idea to use a framework which can assist in development and
that can make the setup process easier. Similarly, when building your own hooks, you need to have a solidity framework
in pace and then you will also need to use a few utilities to 
1) Mine the correct salt for your hook address
2) Using a router which acquire the lock and then call various functions on the pool manager
3) Deploying the hook to a deterministic address

Many of these steps are already taken care by a few template repositories which are listed [here](https://uniswaphooks.com/components/hooks/templates)

## Hooks Template 
One of the most popular templates is the [v4-template](https://github.com/uniswapfoundation/v4-template) which is 
created by [Sauce](https://github.com/saucepoint).

You can clone the repository and then follow the instructions in the README to get started.

## Getting Started
1. **Installation**: The template requires Foundry. Install dependencies using:
   ```
   forge install
   ```
2. **Testing**: Run tests with:
   ```
    forge test
   ```
## Counter Hook
The template includes an example hook (`Counter.sol`), a test for it, and scripts for deploying hooks.

Here are some of the things to note about the Counter hook:

1. It extends the `BaseHook` contract which is defined in the [v4-periphery](https://github.com/Uniswap/v4-periphery/blob/main/contracts/BaseHook.sol) repository.
   ```solidity
    import {BaseHook} from "v4-periphery/BaseHook.sol";
    
    contract Counter is BaseHook {
        // ... hook code here
    }
      
   ```
2. The hook gets called before and after every swap and modify position call. This is done as part of `getHooksCalls` function

   ```solidity
   function getHooksCalls() public pure override returns (Hooks.Calls memory) {
        return Hooks.Calls({
            beforeInitialize: false,
            afterInitialize: false,
            beforeModifyPosition: true,
            afterModifyPosition: true,
            beforeSwap: true,
            afterSwap: true,
            beforeDonate: false,
            afterDonate: false
        });
    }
   ```

3. Corresponding to the hooks calls, the hook implements the following functions:
    ```solidity
    function beforeSwap(
        address sender,
        SwapInfo calldata swapInfo,
        bytes calldata data
    ) external override returns (bytes memory) {
        // ... hook code here
    }
    
    function afterSwap(
        address sender,
        SwapInfo calldata swapInfo,
        bytes calldata data
    ) external override returns (bytes memory) {
        // ... hook code here
    }
    
    function beforeModifyPosition(
        address sender,
        ModifyPositionInfo calldata swapInfo,
        bytes calldata data
    ) external override returns (bytes memory) {
        // ... hook code here
    }
    
    function afterModifyPosition(
        address sender,
        ModifyPositionInfo calldata swapInfo,
        bytes calldata data
    ) external override returns (bytes memory) {
        // ... hook code here
    }
   ```
4. In each of these functions, Counter hook increments a counter. You can see this in the `beforeSwap` function:
   ```solidity
    function beforeSwap(address, PoolKey calldata key, IPoolManager.SwapParams calldata, bytes calldata)
        external
        override
        returns (bytes4)
    {
        beforeSwapCount[key.toId()]++;
        return BaseHook.beforeSwap.selector;
    }
   ```

## Additional Resources
- **v4-periphery**: For advanced hook implementations.
- **v4-core**: The core repository for Uniswap v4.




