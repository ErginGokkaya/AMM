# Uniswap V4 Hooks Course
## History from v1 to v4
- v1: It has a factory contract allowing anyone to deploy a pair contract to swap ETH and an ERC20 token. This limitation was less attactive for the users.
- v2: It introduces ERC20 to ERC20 pools, multihop routing strategies for swaps, flash swap and TWAP price oracles. Main problem which _Liquidity Providers (LPs)_ faced is **impermanent loss**. Also, LPs have to provide liquidity across the entire price range even though swappers gather for a certain range.
- v3: It introduces the concentrated liquidity which enables the LPs to provide liquidity for a certain range.
- v4: The most important novelty is the hooks. Hooks allow anyone to write smart contracts that can modify the behaviour of a liquidity pool by "hooking into" key parts of the control flow. This allows anyone, effectively, to create derivate DEXes on Uniswap without needing to fork the entire codebase. For gas optimization, v4 defines transient storage.

By creating hooks that modify the control flow of the DEX, a ton of exciting ideas become possible.

## v4 Architecture
### How was it in UniswapV3?
Each pool contract needs to be deployed by a factory contract. This means all the pools are seperate and they don't share any liquidity each other.
### UniswapV4 Singleton Design
The main idea behind is:  Instead of pools being their own contracts, a single contract now manages all pools. To do that, a pool manager contract defined. Then all the pools are managed by a single instance of thee pool manager contract. This design is possible because pools now are implemented as Solidity libraries.
Unlike V3 pool mappings pointing to a contract address, in v4 the pool mappings points to a _Pool.State_ struct. All the AMM logic gets encapsulated within the PoolManager contract directly requiring no external function calls using this Pool library struct.
In Uniswap v3 - view functions like slot0() existed on each Pool contract to return some crucial information:
```Java
interface IUniswapV3PoolState {
	function slot0() external view returns (
		uint160 sqrtPriceX96,
		int24 tick,
		uint16 observationIndex,
		uint16 observationCardinality,
		uint16 observationCardinalityNext,
		uint8 feeProtocol,
		bool unlocked
	);
}
```
v4 singleton design made possible by offloading most of the logic to libraries and calling functions through those libraries rather than deploying new contracts entirely for each pool.

### Flash Accounting
In previous versions of Uniswap, every time a swap was made - including multi-hop swap - output tokens had to be transferred out the Pool contract and then transferred into whatever their destination was (next pool in case of multi-hop, or back to the user in case of simple swap). This design led to inefficiencies because moving tokens around by calling external functions on their smart contracts - especially in a multi-hop swap - was quite expensive. But also, this was the only design possible since each pool was it's own contract and we needed to move tokens in and out of them to keep the logic sound.
With the singleton architecture, a better design now exists - this is called Flash Accounting.

### Locking
To access pool manager, you need to unlock the functions first. To do that, you can call unlock function which triggers a callback function like `IUnlockCallback(msg.sender).unlockCallback(data);`. Then you can implement the code inside this callback. Once the desired execution finishes, callback function returns. Then pool manager check the delta balances to make sure there remains no pending debits or credits. And finally, unlock function lock the manager again.

### Transient Storage
EIP 1153 introduces two new OPCODES to the EVM, TSTORE and TLOAD. This storage only exists in the context of a single transaction and does not persist after the transaction is complete.
> Example:
Whether or not the PoolManager is currently locked is simply a boolean value tstore'd at a specific memory location corresponding to uint256(keccak256("Unlocked")) - 1 in the EVM.

Currently, tstore and tload can only be used when writing low-level assembly in Solidity as we do not have higher level functions for them yet. 
```Java
function lock() internal {
        uint256 slot = IS_UNLOCKED_SLOT;
        assembly {
            tstore(slot, false)
        }}
```
### ERC-6909 Claims
ERC-6909 is a simple multi-token contract standard. Think of it as a way to manage multiple ERC-20 tokens from a single smart contract - which improves upon ERC-1155's design by actually simplifying it. For v4, ERC-6909 is used to represent claim tokens.
## Hooks
A hook is a smart contract that you write that can be attached to liquidity pools to make that pool have custom behaviour by plugging into key points of different actions.
### Hook Address Bitmap
Each hook function - for e.g. beforeInitialize - corresponds to a certain "flag". For example, the beforeInitialize function is correlated to the **BEFORE_INITIALIZE_FLAG** which has a value of 1 << 13
These flags represent specific bits in the address of the hook smart contract - and the value of the bit (a zero or a one) represents whether that flag is set to true or false.
> Example:
Take an address like: 0x00000000000000000000000000000000000000**90** which is in bits: 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 **1001 0000**. The AFTER_DONATE flag is represented by the 5th bit - which is set to 1 for the given contract address. That means this contract address - if it is a hook contract address - signals that the afterDonate function is implemented in it and the PoolManager will attempt to call that function during a donate workflow. Similarly, the 8th bit which is also a 1, actually corresponds to the BEFORE_SWAP i.e. the beforeSwap hook function - which will also attempt to be called by the PoolManager during a swap workflow.

Now - of course - addresses can be deceiving and lie about their implementations. In that case, initializing a pool with a "fake" hook will simply fail transactions which attempt to call into unimplemented hook functions. Therefore, when deploying hooks to an actual network - we need to do a little bit of address mining to find a proper deployment address for our hook which can properly represent the functions we have implemented in our hooks.
For more details about hooks' flags, refer to: `https://github.com/Uniswap/v4-core/blob/main/src/libraries/Hooks.sol`.

**!!Notice:** Balance delta retruns two int values, amount0 and amount1. A negative value tells transfer required from user to pool manager. Similarly, a positive value tells transfer from pool manager to user. These balances can be settled in one of two ways:
1. Actually transferring tokens in both directions
2. Minting/burning ERC-6909 tokens the user has from previously locked up money in the PoolManager

Therefore - no tokens actually are transferred until the very end of the transaction flow, right before the callback finishes executing. This allows multi-hop swaps to only require token transfers (or 6909 minting/burning) for the initial input and final output tokens without needing to worry about the "in-between" steps since those deltas just get zero'd out.

### Liquidity Position Modification Flow
It is almost same with the flow for swap operation. Periphery contract unlocks the PoolManager which then does a callback to unlockCallback. Inside the callback, the periphery contract triggers modifyLiquidity on the manager. The manager ensures the pool is valid and is initialized. Manager figures out if the modification is an addition or removal of liquidity. Manager checks if the associated before hook needs to be called based on the hook's address - and calls it if necessary. The actual liquidity modification logic runs and the balance delta is calculated. The ModifyLiquidity event is emitted. The manager checks if the associated after hook needs to be called, and calls it if necessary. The balance delta is returned to the periphery contract. The periphery contract settles all balances (by transferring tokens or minting/burning 6909 claims). unlockCallback finishes execution, Manager ensures no pending non-zero deltas, and locks itself. Transaction is complete.

### Liquidity Fragmentation
A pool in v4 has a unique identifier consisting of: token 0, token 1, hook address, tick spacing, swap fees. Therefore, it is possible (and in fact reasonable) to have multiple pools present for the same token pair. 

## Pool State
Unlike previous versions, v4 uses a different approach for storing and accessing pool data, which requires understanding the use of StateLibrary and extsload. Pools are managed by mapping id to a struct (`mapping(PoolId id => Pool.State) internal _pools;`).
```java
contract PoolManager {
    using Pool for Pool.State;
    mapping(PoolId => Pool.State) internal pools;
 
    function swap(PoolId id, ...) external {
        pools[id].swap(...); // Library call
    }
}
```
**getPoolState(PoolKey calldata key)** returns slot0 parameters. 
## Uniswap Math
To understand the math behind, you need to know what tick and tick spacing are in both uniswapv3 and uniswapv4. In the diagram below, the numbers on the line - -9 to +9 - are what are called "ticks". Tick space -one for this diagram- is smallest possible price movement possible for these tokens relative to each other.

![Açıklama metni](https://euyfsxp8ng.ufs.sh/f/UFJ0VfqoNCvA4WrQ1R0VrLPMybxn9BGgkNlepU2TC01IdqEJ)

> Example:
A user has x = 2 ETH. They want to create a liquidity position in an ETH/USDC pool. The current price of ETH is P = 2000 USDC. They want to add liquidity in the range of P_a = 1500 to P_b = 2500 USDC. How much USDC do they need?
L_x = x * (sqrt(P ) * sqrt(P_b) / (sqrt(P_b) - sqrt(P )))
y = L_x * (sqrt(P ) - sqrt(P_a))
When we do this, we find out y = 5076.1 USDC - which is the amount of USDC we need.

As you can see there are many sqrts. This causes to need of handling a lot of floaiting points. But Solidity don't have it. To do this, Q numbers are declared.
Q64.96 numbers are a way of representing rational numbers that uses 64 bits to represent the integer part, and 96 bits to represent the fractional part of the given number. A number can be converted between its decimal notation and its Qn.k Notation with a fairly simple equation where D_n is decimal notation of the number:
```
Q_n = D_n * (2^k) where k = 96
```
## Dynamic Fees and Nezlobin Directional Fees
Easier to understand with examples and diagrams. So, refer to original course page: https://learn.atrium.academy/course/9a4ba933-4bee-42fe-871c-6c28b5ca9ffd/dynamic-fees

## Limit Orders
Limit order is an order which is placed at a different price than the current price.
