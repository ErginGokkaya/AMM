# UNISWAP v3

## Difference From UniswapV2
- Imagine a pool of 100 DAI and 100 USDC. In this case 1 DAI = 1 USDC
If want to swap 1 DAI for 1 USDC, the pool become 101 DAI and 99 USDC. This means that 1 DAI = 0.98 USDC
If we want a pool in which 1 token swap causes at most 0.01 change in price (0.99 - 1.01), We need to have more amount of tokens. To satisfy this condition, the pool must start with 200 DAI and 200 USDC. In this case 1 token swap results in inside the interval (0.99 - 1.01).
These amount of tokens are required in uniswapv2 to meet the condition. However, uniswapv3 defines _concentrated liquidity_ to satisfy the condition with much less amounts of tokens. 
(IMHO: This enables liquidity providers to get more profit with the same amount of liquidity when compared to uniswapV2. From previous notes, The bigger the L is, the bigger the swap returns, the less price impact. When the liquidity is low, even a little amount of token swap, changes the price a lot.)

- Take this example into consideration. In UniswapV2, to satisfy the interval (0.99 - 1.01), pool has to have 200 DAI and 200 USDC. Whereas in UniswapV3, the pool needs a lot less than those amounts.
To compare the difference, if you want to get the L-curve created by 200 DAI and 200 USDC in UniswapV3, you have to have a pool of 40100 DAI and 40100 USDC in UniswapV2 so that the pool satisfy the 0.01 change condition. As a result, this means that the liquidity amplified 40100 / 200 fold in UniswapV3 compared to UniswapV2.

Naming in UniswapV3 for this amounts are: 200 is real reserves and 40100 is virtual reserves.

##### Uniswap V2: 
>It tracks the reserves of token X and token Y. Then it calculates the liquidity L^2 = x * y and the price P = y / x .
>
>Passice liquidity management - ERC20 : a liquidity provider just deposits some tokens. over time your liquidity collects swap fees. 
>
>Swap fees are constant 0.3%
>
>TWAP is aritmatic mean
>
>When adding liquidity, you need to deposit both tokens in the pair.

##### Uniswap V3:
>It tracks the liquidity L and the price P. Then it calculates the reserves x and y between the price range P_A nad P_B. x = L / sqrt (P_A) - L / sqrt (P_B) and y = L / sqrt (P_B) - L / sqrt (P_A)
>
> Active liquidity management - ERC721 : as a liquidity provider in Uniswap V3, you have to specify the price range. When the current price is within that specified price range, your liquidity position collects swap fees. otherwise your liquidity position no longer collects a fee. To keep collecting fees, you need to actively engage with the pool according to current price.
>
>Varying swap fees: 0.01, 0.05, 0.3, 1% 
>
>TWAP is geometric mean
>
>Single side liquidity. You can deposit only one of the tokens by specifying the range.

## Pros and Cons of UniswapV3
- Higher capital efficiency for LP
- Active liquidity management
- Liquidity is represented by Nonfungible Tokens - ERC721

# Tick Math
the price in UniswapV3 pool can be calculated by the formula
```
p = 1.0001^t
```
t is the tick number in the range -inf < t < inf.

For negative t values price p<1

For t = 0, price = 1

For t > 0, price > 1

Increasing t corresponds to moving upper left on the L-curve on a constant product graph.

As the price moving to P_A, the liquidity gets turning to token X. And when the price passes P_A, all the liquidity turns to token X. Similarly, As the price moving to P_B, the liquidity gets turning to token Y. And when the price passes P_B, all the liquidity turns to token Y.
Between P_A and P_B, the amount of token X and the amount of token Y changes according to the price (that is tick,t).

In UniswapV2, we mainly utilize the amount of tokens graph. In x and y axises, we represent the amount of tokens X and Y. However, in UniswapV3 this is not much benefical for us. Instead, we use tick (price) - liquidity graph. This graph has tick values on x-axis representing the tick values for prices. For example, price ranges corresponds to tick ranges on x-axis. Besides, the y-axis represents the liquidity.

## Some important contracts
NonFungiblePositionManager : manages your position when you add/remove liquidity and collects fees

SwapRouter

UniswapV3Factory

UniswapV3Pool

SwapRouter02

UniversalRouter

## Spot Price
You can get the spot price from slot0 strcut inside uniswapv3pool contract. This struct has parameters like sqrtPrice, tick. The spot price can be calculated by using them.
Example: X is token 0 and Y is token 1. P is the price of X in terms of Y (in UniswapV2 it is P= Y/X)

P = 1.0001^tick
    or
P = ( sqrtPricex96 / 2^96 )^2

```
!! Take always decimals into consideration
```

## Math Tricks

X = L / sqrt(P)

Y = L sqrt(P)

For a price P, the amount of X = x_r + x_v where x_r is the real value and x_v is the virtural value. x_r is the difference between higher price x and the current price x.
For a price P, the amount of Y = y_r + y_v where y_r is the real value and y_v is the virtural value. y_r is the difference between the current price y and lower price y.
Here the higher price is P_b and the lower price is P_a. They determines the price range.

The curve of real reserves (which is the one that obtained by mobing the original virtual curve to the origin.) touches P_b on the y-axis and P_a on the x-axis. The equation:

(x_r + L/sqrt(P_b)) * (y_r + L*sqrt(P_a)) = L^2

# SWAP
- zeroForOne flag gives the direction of the swap. It is *true*, when you give token 0 to get token 1, and it is *false* otherwise.
- amountSpecified may be positive or negative.
- There are two types of swap, exactInput and exactOutput. If amountSpecified >= 0, the swap is exactInput. The amount here is that the amount the caller specified to put in. On the other hand, exactOutput is for amountSpecified < 0. This means that caller specifies the amount which he wants to get out of the swap.
- swap algorithm applies a while loop until the amountSpecified fulfills or the current price reaches sqrt(P_limit)
- feeGrowth is updated in swap function. It is later used to calculate the swap fee.
- After all the executions done, swap function calls a callback funtion from a msg.sender. this means that msg.sender must be a contract to have callback function.

> !! Homework: what if no one specifies a certain range for a pair, then how would uniswapv3 work? In other words, Let's consider a DAI/USDC pool, if no one provides liquidity for the (0.95,0.96) range, how does uniswapv3 handles this?

- How does UniswapV3 keep track the active liquidity? The active liquidity is the sum of all the liquidity of active price range. Mark the liquidity edges with the liquidity porvided. Let the right edge negative sign and left edge positive sign. Define a liquidity net by summing all intersected liquidity values. Consider the price direction as well. If price is increasing typically use the sign assigned before. else, change the signs before adding.

- What would the new price be after adding/removing liquidity? To do this, delta price is defined. The delta price is nothing but adding a delta sign as a prefix to the amounts(reserves) in the first equations defined for transitions from liquidity and price to reserves. Remaining part is just mathematical manipulations. Only consider that adding delta_x decreases the price, removing delta_x increases the price, adding delta_y increases the price, removing delta_y decreases the price. If this is hard to follow, just imagine the visual constant product curve.

## Swap Fee
Consider a case: lower bound: P_a and upper bound: P_b... amount in is token X and amount out is token Y. current price is P.. user wants to swap his token x to get token y.
A    : initial amount of amount in (token x)

fee  : A*f where 0<= f <= 1

a_in : amount_in after swap fee = A - fee <= max amount in (visualize the graph P_b - Pa)

a_out: amount_out <= max amount out (P - Pa)

## Swap Types
There are 4 types of swap which can be called by the user.
For multi hop swap the path format: path = [token,fee,token,fee,token,...,fee,token]

!! Important: fees in the path are actually set real fee * 1e6

### ExactInputSingle
user calls SwapRouter02 contract, then it calls UniswapV3Pool contract.
Here the path is simple.
path = [token in , fee , token out ]

User -> exactInputSingle -> (router) swap -> (pool) transfer token out to user -> (pool) callback from router -> (router) in callback, transferFrom(user,pool,token in)
- Notice that token out transfered before the user transfer token in to the pool.
### ExactInput
User calls this for multihop swap. Let's say there are 3 pool for swapping token A/B, B/C, C/D. If user want to swap A for D, router executes a swap chain. 
path = [A, fee, B, fee, C, fee, D]

To loop the path, router is set receipient for intermediate swap executions. Before the loop, the payer is the user as expected. Inside the loop, for each swap, function call stack is similar. 
>exactInput (EOA, User, payer)
>
>swap A/B (Router calls swap inside of Pool)
>
>transfer B to receipient from pool (Pool transfers to receipient)
>
>uniswapV3swapcallback              (Pool calls callback function inside of Router)
>
>    transferFrom payer to pool A/B amount of A (Callback of Router calls transferFrom payer the amount to the pool)
>
>update path by extracting A (Router)
>
>set router as payer  (Router)
>
>swap B/C
>
>so on so forth

### ExactOutputSingle
If you want to get some certain amount of token from a swap, yo can use this function in case of single pool swap.
path = [token out, fee, token in]

### ExactOutput

Similar to the previous example, if user wants 100 of token D, he wonders how much amount of token A he should pay.

!! To test the contract in foundry, DONT FORGET to approve the router so that it can spend your deposite token.

# Factory Contract
UniswapV3 uses create2 to deploy the pool contract. This enables the contract to determine the address of the pool contract before deployment. 

create2 -> keccak256(0xFF, address of the deployer, salt(random 32 bytes), keccack256(contract creation bytecode, constructor inputs)) -> returns(addres of the contract(last 20 bytes of the obtained hash))

deployer: factory contract address
salt    : keccak256(token0,token1,fee)
creation bytecode: byte code of pool contract (constant)

# Liquidity
How to determine the amount of token X and token Y to add liquidity for a given price range?
How to determine the amount of token X and token Y to get when removing liquidity?

It has a simple formula resulting in L, liquidity when amount of token X and token Y, current price P, lower price bound P_a and upper price bound P_b are given.

When the current price is less than the P_a, all the liquidity is in token X. While the price moves P_a to P_b, all the liquidity gets turning to token Y step by step. After the price passes the P_b, then all the liquidity is in token Y.

Liquidity calculation is easy when the price is out of the range. Hovewer, when the price in in the range, you need to calculate L_x and L_y individually. L_x is the liquidity between P_b and P whereas L_y is the liquidity between P and P_a.

- Liquidity Delta
L0: liquidity before
L1: liquidity after
Consider these both as before/after of adding or removing liquidity
delta_L = L1 - L0
Q: How much delta_x and delta_y to add or remove for delta_L changes in liquidity between range P_a and P_b?

- The price range boundries must be multiplies of tick spacing. Lower tick spaces concantrates the liquidity higher at the cost of using more gas for a swap.

## Mint Function
Inputs: receipient, tickLower, tickUpper, amount, data

# Tick Bitmap
It is a mapping from int16 to uint256
tick value is stored as int24 in slot0. It has two parts. First part is (from left) int16 and the other in uint8.
ex: if tick = -200697.. then first part (word position) is -784.. and second part is (bit position) 7..

!! I don't understand why this is needed. It is a simple bit manipulation. But what the advantage is? No idea.
I totally understand nothing why this is calclualted like that.

# TWAP
Contrary to UniswapV2, TWAP is calculated by geometric mean. 
Let's say prices are P_t, P_(t+1), P_(t+2),..., P_(t+n)

P_TWAP = (P_t * P_(t+1) * P_(t+2)...* P_(t+n))^(1/n)

tickCumulative stores the price multiplication until so far.

- P_x : spot price of token X in terms of token Y
P_y : spot price of token Y in terms of token X

Both in UniswapV2 and UniswapV3 this is valid: P_x = 1 / P_y

However, this relation is not valid for TWAP price in UniswapV2. But valid in UniswapV3.

TWAP of UniswapV3 converges the spot price spikes down (huge price decreases) faster than TWAP of UniswapV2. (Not sure whether this is good or not). However, it is slower for spikes up (huge price increases). 
