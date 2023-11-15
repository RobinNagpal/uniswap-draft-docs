## Testing hooks
Testing hooks is very similar to testing contracts. The template includes a test for the Counter hook, which you can find in `test/Counter.t.sol`.

Here are some key points about the Counter hook test:
1. The hook extends from several utilities that facilitate easier testing of hooks.

    ```solidity
        import {HookTest} from "./utils/HookTest.sol";
        import {Deployers} from "@uniswap/v4-core/test/foundry-tests/utils/Deployers.sol";
        import {GasSnapshot} from "forge-gas-snapshot/GasSnapshot.sol";
        
        contract CounterTest is HookTest, Deployers, GasSnapshot {
        }
    ```

2. The `setup` function, called before every test, creates a few test tokens, retrieves the hook address, and then initializes the pool with this hook address. 

    ```solidity
        function setup() public {
            // creates the pool manager, test tokens, and other utility routers
            HookTest.initHookTestEnv();
    
            // Deploy the hook to an address with the correct flags
            uint160 flags = uint160(
                Hooks.BEFORE_SWAP_FLAG | Hooks.AFTER_SWAP_FLAG | Hooks.BEFORE_MODIFY_POSITION_FLAG
                    | Hooks.AFTER_MODIFY_POSITION_FLAG
            );
            (address hookAddress, bytes32 salt) =
                HookMiner.find(address(this), flags, type(Counter).creationCode, abi.encode(address(manager)));
            counter = new Counter{salt: salt}(IPoolManager(address(manager)));
        }
    ```
    Pool is then initialized with this hook
    ```solidity
        // Create the pool
        poolKey = PoolKey(Currency.wrap(address(token0)), Currency.wrap(address(token1)), 3000, 60, IHooks(counter));
        poolId = poolKey.toId();
        manager.initialize(poolKey, SQRT_RATIO_1_1, ZERO_BYTES);

    ```
   
3. Hook tests utilize a router, namely `PoolModifyPositionTest`, to modify positions. PoolModifyPositionTest implements the `ILockCallback` interface and adds the `lockAcquired` function, which in turn calls the `manager.modifyPosition` function. 
   ```solidity
        PoolManager manager;
        PoolModifyPositionTest modifyPositionRouter;
   
        manager = new PoolManager(500000);

        // Helpers for interacting with the pool
        modifyPositionRouter = new PoolModifyPositionTest(IPoolManager(address(manager)));
    
        modifyPositionRouter.modifyPosition(poolKey, IPoolManager.ModifyPositionParams(-120, 120, 10 ether), ZERO_BYTES);
   ``` 
   Similarly, for token swaps, the test uses `PoolSwapTest`, which also implements the `ILockCallback` interface.

4. Testing the hook closely resembles testing any other smart contract. The function `testCounterHooks` executes swaps and verifies if the counters are updated correctly.
    
   ```solidity
        function testCounterHooks() public {
          // Perform a test swap //
          int256 amount = 100;
          bool zeroForOne = true;
          swap(poolKey, amount, zeroForOne, ZERO_BYTES);
          // ------------------- //

          assertEq(counter.beforeSwapCount(poolId), 1);
          assertEq(counter.afterSwapCount(poolId), 1);
        }
    ```
