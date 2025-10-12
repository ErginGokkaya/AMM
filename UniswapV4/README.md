# UNISWAP V4 NOTES

Contents
    - PoolManager
    - PositionManager
    - UniversalRouter
    - Hooks

## Difference Between UniswapV3 and UniswapV4
- Hooks
- Dynamic Fees
- Singleton Pattern
- Flash accounting
- New token standar ERC6909

> By singleton design, all the pair pools are inside a single contract. This enables multihop swaps with less gas usage. In UniswapV3, a multihop swap is executed by transfering the intermediate tokens to router. Then router manages the flow to get the final desired token. However in UniswapV4, because all the pools are inside a single contract (same contract), no need to transfer the intermediate tokens.

## New data types
Currency: just an alias of the type address
PoolKey : is a struct that stores (Currency) currency0 and currency1, fee, tickSpacing, (IHook) hooks
PoolId  : keccak256(poolKey, 0xa0).. ID of a pool.. 0xa0 : size of the PoolKey

!! Address of native ETH is address(0)

## Swap
To swap, user must call unlock function to unlock the swap function. the *unlock* function triggers a callback function defined in msg.sender. This means an EOA (regualr user) cannot directly interact with PoolManager contract.
After the callback is triggered, caller contract (msg.sender) implements its own functionality.

> there are new opcodes in Solidity for transient storage. tstore, tload. A transient storage stores a variable during a transaction. Advantages: it uses less gas, resets values after each transactions. Main usecases: reentrancy lock, callback contexts

## Account delta
Router -> PoolManager : swap 1000 USDC for 0.2 ETH

| Action  | Target | Currency | Account Delta |
| ------  | ------ | ------ | ------ |
| unlock  | Router |        |        |
| swap    | Router |  USDC  | -1000  |
|         |        |  ETH   |  0.2   |
|  take   | Router | USDC   |  -1000 |
|         |        |        |   0    |
| settle  | Router | USDC   |   0    |
|         |        | ETH    |   0    |

## Hooks
(Official Definition) : Hooks are external contract calls that are made from the pool manager contract before/after major operations such as initializing a pool, modifying liquidity and swaps. There are functions including the prefixes before- and after- in PoolManager.sol

All hook functions are declared in IHook.sol interface.

PoolManager knows what hooks are gonna used by its PoolKey parameter. PoolKey Struct has an argumnet for IHook hooks.

```
Reminder: 
PoolKey strcut has currency0, currency1, fee, tickSpacing, hooks
PoolID = keccak256(poolKey, 0x(size of poolkey))
```

Even if all the parameters are same except hooks, a different poolId created due to the keccak256 hash.
This means that there might be many different pools with same parameters and different hooks.
As a result, each pool may have only one hook contract and each hook contract may have various pools. 

Example: Lets say Hook1 with WBTC/USDC with poolId 0x1111..111 (32 bytes)
Hook2 with USDC/USDT with poolId 0x222..222
Hook3 with USDC/USDT with poolId 0x333..333
Hook3 with WBTC/USDT with poolId 0x444..444 ( may differ to fee or tick spacing)
Hook3 with SOL/USDT  with poolId 0x555..555

We have 3 pools with hook3. 

- A sample function call stack
User calls swap from Router
Router calls unlock from UniswapV4 PoolManager
PoolManager calls a unlockCallback from Router
Router calls swap from PoolManager
PoolManager calls afterSwap from Hook with parameter sender = Router
? How does hook know the user? Hook's msg.sender is PoolManager. Only Router contract has msg.sender is User address.
Hook may call a getMsgSender function of sender (router).


