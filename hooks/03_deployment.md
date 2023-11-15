# Deployment
Deploying Uniswap v4 Hooks involves several steps:

1. **Deploying the PoolManager Contract**: This contract is typically pre-deployed on many test environments. However, 
you have the option to deploy it locally on your machine if required.

2. **Deploying the Hook Contract**: The hook contract needs to be deployed at a predetermined address. You can use 
`CREATE2` for deterministic deployment. A Deterministic Deployment Proxy, usually found 
at `0x4e59b44847b379578588920cA78FbF26c0B4956C`, is employed for this purpose and is already available in most 
environments.

3. **Deploying Test Tokens**: These tokens are essential for creating the pool. They need to be deployed before 
initializing the pool.

4. **Initializing the Pool with the Hook Contract Address**: This is achieved by invoking 
the `initialize(PoolKey memory key, uint160 sqrtPriceX96, bytes calldata hookData)` function on the PoolManager contract.

5. **Adding Liquidity or Modifying Position**: If you wish to add liquidity to the pool or alter its position, a 
utility contract that implements the `ILockCallback` interface is necessary. You may consider deploying a utility 
contract like `PoolModifyPositionTest` for these operations.


## Deployment Scripts
The template includes a few scripts that help with deploying hooks. These scripts are located in the `scripts` folder.

Lets look at these scripts one by one:

#### 1. Deploying Your Own Tokens
The template includes Mock UNI and Mock USDC contracts for testing. Deploy them using:
   ```
   forge create script/mocks/mUNI.sol:MockUNI \
   --rpc-url [your_rpc_url_here] \
   --private-key [your_private_key_on_goerli_here] \

   forge create script/mocks/mUSDC.sol:MockUSDC \
   --rpc-url [your_rpc_url_here] \
   --private-key [your_private_key_on_goerli_here] \
   ```


#### 2. script/PoolCreateSolo.s.sol
This script contains the steps for initializing the pool without a hook. It uses the pre-deployed PoolManager contract and 
token contracts 

```solidity
    address constant GOERLI_POOLMANAGER = address(0x3A9D48AB9751398BbFa63ad67599Bb04e4BdF98b); // pool manager deployed to GOERLI
    address constant MUNI_ADDRESS = address(0xbD97BF168FA913607b996fab823F88610DCF7737); // mUNI deployed to GOERLI -- insert your own contract address here
    address constant MUSDC_ADDRESS = address(0xa468864e673a807572598AB6208E49323484c6bF); // mUSDC deployed to GOERLI -- insert your own contract address here
    address constant HOOK_ADDRESS = address(0x0); // hookless pool is 0x0!
```
Here it uses the pre-deployed PoolManager contract and token contracts. You can replace these addresses with your own.

It then initialized the pool without the hook

```solidity
    bytes memory hookData = new bytes(0);

    PoolKey memory pool = PoolKey({
        currency0: Currency.wrap(token0),
        currency1: Currency.wrap(token1),
        fee: swapFee,
        tickSpacing: tickSpacing,
        hooks: IHooks(address(HOOK_ADDRESS)) // 0x0 is the hookless pool
    });

    manager.initialize(pool, startingPrice, hookData);

```

#### 3. scrips/CounterDeploy.s.sol
This script deploys the Counter hook using Deterministic Deployment Proxy. It uses the pre-deployed PoolManager contract
and proxy

```solidity
    // hook contracts must have specific flags encoded in the address
    uint160 flags = uint160(
        Hooks.BEFORE_SWAP_FLAG | Hooks.AFTER_SWAP_FLAG | Hooks.BEFORE_MODIFY_POSITION_FLAG
            | Hooks.AFTER_MODIFY_POSITION_FLAG
    );

    // Mine a salt that will produce a hook address with the correct flags
    (address hookAddress, bytes32 salt) =
        HookMiner.find(CREATE2_DEPLOYER, flags, type(Counter).creationCode, abi.encode(address(GOERLI_POOLMANAGER)));

    // Deploy the hook using CREATE2
    vm.broadcast();
    Counter counter = new Counter{salt: salt}(IPoolManager(address(GOERLI_POOLMANAGER)));

```
You can deploy your own hook by using the following command
```
forge script script/CounterDeploy.s.sol \
--rpc-url [your_rpc_url_here] \
--private-key [your_private_key_on_goerli_here] \
--broadcast
```


#### 4. scrips/PoolCreateHook.s.sol
This script contains the script for initializing the pool with the hook. It uses the pre-deployed PoolManager 
contract and hook addresss.

Note: Update the hook address to your own hook which you get from the previous step.

```solidity
    address constant HOOK_ADDRESS = address(0x3CA2cD9f71104a6e1b67822454c725FcaeE35fF6); //address of the hook contract deployed to goerli -- you can use this hook address or deploy your own!
    address hook = address(HOOK_ADDRESS);

    PoolKey memory pool = PoolKey({
        currency0: Currency.wrap(token0),
        currency1: Currency.wrap(token1),
        fee: swapFee,
        tickSpacing: tickSpacing,
        hooks: IHooks(hook)
    });

    manager.initialize(pool, startingPrice, hookData);
```
