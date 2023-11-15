# Intro to Locking
The locking mechanism in v4 ensures that certain operations are executed atomically without interference, ensuring 
consistency and correctness in the PoolManager's state. PoolManager, uses `LockDataLibrary` to manager a queue of 
lockers, allowing nested locks, and ensures that all currency deltas are settled before releasing a lock.

Pool actions can be taken by acquiring a lock on the contract and implementing the `lockAcquired` callback to 
then proceed with any of the following actions on the pools:

- `swap`
- `modifyPosition`
- `donate`
- `take`
- `settle`
- `mint`

# Simple Locking Analogy

Imagine a public library where people can come in and borrow books. Now, let's say a librarian wants to update the
records for a specific book (e.g., mark it as borrowed). To do so without interruptions, the librarian puts up a
sign that says, "Please wait, updating records." This sign prevents other librarians or staff from making changes
to the same record at the same time. Once the updating is done, the sign is removed, and others can now access and
modify the record.

In this analogy:
- The book record is like the state of the PoolManager.
- The sign the librarian puts up is equivalent to acquiring a lock.
- Other staff waiting or not being able to update the record represents the protection given by the lock.


# Main Components
In `PoolManager` "locking" is essentially a way to ensure certain operations are coordinated and don't interfere with each other.
Here are the main components of the locking mechanism:

### 1. **Locking and Unlocking:**
  - The `lock` function is where the locking mechanism is initiated. It pushes the `msg.sender` (the caller of the 
    function) to the `lockData` which acts as a queue.

      ```solidity
      function lock(bytes calldata data) external override returns (bytes memory result) {
          lockData.push(msg.sender);
  
          result = ILockCallback(msg.sender).lockAcquired(data);
  
          if (lockData.length == 1) {
              if (lockData.nonzeroDeltaCount != 0) revert CurrencyNotSettled();
              delete lockData;
          } else {
              lockData.pop();
          }
      }
      ```

  - During the lock, a callback function `ILockCallback(msg.sender).lockAcquired(data)` is called, where the locked 
    contract can perform necessary operations.

  - After the operations in the callback are completed, it either deletes the `lockData` if it is the only element, 
    signifying the release of the lock, or it pops the last element from `lockData`, signifying that the lock is 
    released by that particular address.

### 2. **Queue Management:**

  - The queue management is handled by the `LockDataLibrary`. This library provides functionality to push, pop, and 
    get active locks from the `lockData`.

  - The queue is managed in a way that it allows you to keep a track of locker addresses and ensure that only the 
    active locker can perform certain operations.

### 3. **Non-Zero Deltas Tracking(Flash Accounting):**

  - `nonzeroDeltaCount` is a variable in `LockData` structure that keeps track of non-zero deltas across all lockers. A 
     delta here appears to represent a kind of balance or a state that changes during operations.

  - In the `_accountDelta` function, if a delta changes from or to zero, the `nonzeroDeltaCount` is updated accordingly. 
    This is crucial for tracking the net changes made by each locker and ensuring that everything nets to zero at the end of operations.

  - This mechanism is a key component of what is termed "Flash Accounting" in Uniswap V4. Flash Accounting is an innovative approach introduced with the new singleton-style pool management. This feature fundamentally alters the management of tokens during transaction processes. Traditional methods typically require explicit tracking of token balances at every operational phase. In contrast, Flash Accounting operates under the principle that by the end of each transaction or "lock period," there should be no net tokens owed either to the pool or the caller, streamlining the accounting process significantly.

  - Flash accounting leverages the capabilities of transient storage opcodes, as proposed in EIP-1153. These transient storage opcodes are incredibly efficient, acting like temporary variables that exist only for the duration of the transaction. This dramatically reduces the gas costs because you're not continually updating the Ethereum state with intermediate steps.  
### 4. **Restricted Access:**

  - The modifier `onlyByLocker` is used to restrict access to certain functions. It ensures that a function can only be 
    called by the address that currently holds the lock.
      ```solidity
      modifier onlyByLocker() {
          address locker = lockData.getActiveLock();
          if (msg.sender != locker) revert LockedBy(locker);
          _;
      }
      ```
  - Several functions in the contract, such as `modifyPosition`, `swap`, `donate`, `take`, `mint`, and `settle`, use 
      the `onlyByLocker` modifier. This ensures that only the entity that has currently locked the contract can modify 
      positions, swap tokens, etc. This helps ensure operations are atomic and avoid potential race conditions.

# Locking Diagram

![Locking Diagram](images/05_locking_mechanism/LockingMechanism_excali.png)

* A locker (caller) initiates a lock function execution in the PoolManager contract.
* The locker gets added to the lock queue by the LockDataLibrary.
* An ILockCallback is used where the locker acquires the lock.
* Balances and currency deltas are managed and updated within the Balance and CurrencyDelta.
* The locker can account for pool balance deltas which handle the currency and its corresponding deltas.

### Working
The locking mechanism in the PoolManager contract works as follows:

1. When a user wants to lock, they call the `lock()` function with the data that they want to be passed to the callback.
2. The `lock()` function pushes the user's address onto the locker queue.
3. The `lock()` function then calls the `ILockCallback(msg.sender).lockAcquired(data)` callback.
4. The callback can do whatever it needs to do, such as updating the user's balances or interacting with other contracts.
5. Once the callback is finished, it returns to the `lock()` function.
6. The `lock()` function checks if there are any other lockers in the queue. If there are, it pops the next locker off the queue and calls the callback for that locker.
7. If there are no more lockers in the queue, the `lock()` function returns.

# Lock Data Structure

- **Struct:**
  The `LockData` struct is used to store information related to locks. It has two fields:
   - `length`: Represents the current number of active locks.
   - `nonzeroDeltaCount`: Represents the total number of non-zero deltas over all active and completed locks.

- **Library (`LockDataLibrary`):**
  The `LockDataLibrary` is a library used for managing the custom storage implementation of the queue that tracks current lockers.


# Example
Below is the example from a community ["Liquidity Bootstrapping Hook"](https://github.com/kadenzipfel/uni-lbp/blob/main/src/LiquidityBootstrappingHooks.sol) hook that uses the locking mechanism and calls the `modifyPosition` and `swap` functions.

```solidity
/// @notice Callback function called by the poolManager when a lock is acquired
///         Used for modifying positions and swapping tokens internally
/// @param data Data passed to the lock function
/// @return Balance delta
function lockAcquired(bytes calldata data) external override poolManagerOnly returns (bytes memory) {
    bytes4 selector = abi.decode(data[:32], (bytes4));

    if (selector == IPoolManager.modifyPosition.selector) {
        ModifyPositionCallback memory callback = abi.decode(data[32:], (ModifyPositionCallback));

        BalanceDelta delta = poolManager.modifyPosition(callback.key, callback.params, bytes(""));

        if (callback.params.liquidityDelta < 0) {
            // Removing liquidity, take tokens from the poolManager
            _takeDeltas(callback.key, delta, callback.takeToOwner); // Take to owner if specified (exit)
        } else {
            // Adding liquidity, settle tokens to the poolManager
            _settleDeltas(callback.key, delta);
        }

        return abi.encode(delta);
    }

    if (selector == IPoolManager.swap.selector) {
        SwapCallback memory callback = abi.decode(data[32:], (SwapCallback));

        BalanceDelta delta = poolManager.swap(callback.key, callback.params, bytes(""));

        // Take and settle deltas
        _takeDeltas(callback.key, delta, true); // Take tokens to the owner
        _settleDeltas(callback.key, delta);

        return abi.encode(delta);
    }

    return bytes("");
}
```

Other important thing to note is that before the lock is released, the `nonzeroDeltaCount` is checked to ensure that
all currency deltas are settled. This is done by `_takeDeltas` and `_settleDeltas` functions.

```solidity
/// @notice Helper function to take tokens according to balance deltas
/// @param delta Balance delta
/// @param takeToOwner Whether to take the tokens to the owner
function _takeDeltas(PoolKey memory key, BalanceDelta delta, bool takeToOwner) internal {
    PoolId poolId = key.toId();
    int256 delta0 = delta.amount0();
    int256 delta1 = delta.amount1();

    if (delta0 < 0) {
        poolManager.take(key.currency0, takeToOwner ? owner[poolId] : address(this), uint256(-delta0));
    }

    if (delta1 < 0) {
        poolManager.take(key.currency1, takeToOwner ? owner[poolId] : address(this), uint256(-delta1));
    }
}

/// @notice Helper function to settle tokens according to balance deltas
/// @param key Pool key
/// @param delta Balance delta
function _settleDeltas(PoolKey memory key, BalanceDelta delta) internal {
    int256 delta0 = delta.amount0();
    int256 delta1 = delta.amount1();

    if (delta0 > 0) {
        key.currency0.transfer(address(poolManager), uint256(delta0));
        poolManager.settle(key.currency0);
    }

    if (delta1 > 0) {
        key.currency1.transfer(address(poolManager), uint256(delta1));
        poolManager.settle(key.currency1);
    }
}
```
