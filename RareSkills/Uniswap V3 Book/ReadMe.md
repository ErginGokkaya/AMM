# NOTES FROM UNISWAP V3 BOOK 
## Ch1: Concentrated Liquidity
The reader is already assumed to understand Uniswap V2. Nonetheless, a few term needs to be clarified.
- **Reserve:** A reserve of a token is the amount (balance) of the token held by AMM. In this document x will be denoted as the amount of token X.
- **Liquidity:** In the context of AMM of a pair pool, the liquidity means combined reserves of pair tokens X and Y. Liquidity providers provide liquidity by depositing tokens to the pool and earn fees from swaps or pool growth. In UniswapV2 it is the product of the reserves. (greater sign is because the fees from swaps are added to the reserves)
```
x*y >= k = L^2
```

### Liquidity vs Price
High liquidity enables the traders to swap more tokens. In other words, the higher the liquidity, the more swap available. Furthermore, as the liquidity increases, the swap's price impact reduces. That is, price changes get less with the increasing liquidity. 
The price of a token is the ratio of reserves.
```
price 0 = reserve(token 1) / reserve(token 0)
```
For terminology, the token X and token Y in UniswapV2 change to token 0 and token 1 in UniswapV3. As the reserve of token 0 decreases, its price gets higher. As long as the ratio of reserves remains same, the price remains same regardless of the liquidity.
### Price Impact
Any swap in an AMM moves the price against the swapper. For example, if a trader wants to swap token0 for token1 and deposit token0 to the pool, this makes token1 more expensive. Considering the price equation, this makes much sense. If you add token0 to the pool and take token1 from the pool, the ratio (that is price) changes accordingly. This is called *price impact*.
Result: For a trade of a fixed size, the greater the liquidity, the lower the price impact.
> Example:
Assume a pool of pair 10 USDC and 10 USDT. If you want to buy 1 USDC, reserve of USDC will be 9. To satisfy the constant product formula (x*y=k), reserve of USDT must be 11.11 (9*reserve(USDT)=100). This means the trader must deposit  11.11 - 10 = 1.11 USDT. Then this results in a price increase in USDC. p_USDC = 11.11 / 9 = 1.234, that is a 23.4% price increase.
However, if the pool would have 100 USDC and 100 USDT (k = 10000), 99 USDC * new reserve (USDT) = 10000 and so the USDT reserve would be 101.01. Then the price of USDC would become p = 101.01 / 99 = 1.02, that is only 2%. This means a much less price impact. Also, the trader should pay only 1.01 USDT to get 1 USDC in this case. Previously it was 1.11 USDT.

### Liquidity Concentration
In UniswapV2 liquidity must enable traders to trade in the price range from 0 to infinity. But, most of the time no trade happens for these extreme prices and this wastes the liquidity which provided for them. The idea comes about here. Why don't we concentrate the liquidity to the region we expect the trade happen. As a first step, let's define different sub curves for certain price ranges.
```
        k1, tick0 <= p < tick1
        k2, tick1 <= p < tick2
        .
x*y =   .
        .
        kn, tick(n-1) <= p < tick(n)
```

**Note:** The tick means a predefined price. (p = 1.0001^tick)
If the market price changes significantly, then liquidity providers will remove their liquidity and place it around the new price to capture the swap fees.

## Ch2: The Tick
The liquidity providers can choose segments in the price curve to place their liquidity. For example, in a ETH:USDC pool, if the price of ETH is USDC 2,000, a liquidity provider might choose to place their liquidity between USDC 1,800 and USDC 2,200, so they can capture more fees in the price range.
For the gas usage consumption and the code complexity to reduce, UniswapV3 defines a predefined price points. These points correspond to ticks. The greater the “angle” from the x-axis, the higher the price of asset X in terms of asset Y. So higher the tick value higher the price X. 
The tick may have values from -887.272 to 887.272.
Tick = 0 is theta = 45 degree line (X and Y is 1:1)
Tick < 0 gets closer to x-axis
Tick > 0 gets closer to y-axis
The price range can be defined by upper tick price pb and the lower tick price pa.

The regular price curve can be map to a line. This line gives the tick prices of token X. While the token X price increases (amount of token X reduces), tick price moves left ot right on this line. This line is sliced by the ticks and LPs add liquidity to this tick ranges.

The “current tick” is the current price rounded down to the nearest tick. If the price increases and crosses a tick, then the tick that was just crossed becomes the current tick.

### slot0
The slot0 is a struct including some parameter in the slot 0 in the storage. The current tick is one of the parameter stored in this struct. It is declared by the type int24.
**Note:** Calculating the price of a pair, always consider the number of decimals of the tokens.

## Ch3: Q Number Format
Q number format is a notation for describing binary fixed-point numbers. A Q number multiplies the fraction by a power of 2 instead of a power of 10 because multiplying (and dividing) by powers of 2 is more gas efficient in the EVM, since multiplying or dividing by powers of two can be done using bitshift operations. For example, x << n is equivalent to x * 2** n and x >> n is equivalent to x / 2**n.
A Q number is often written in the form Qm.n where n is the power of 2 used to multiply the number by and m is the unsigned integer size of the integer portion. This is completely equivalent to saying m is the number of bits used to store the integer portion and n is the number of bits used to store the fraction. So the entire number length become m + n. m represents the integer part and n represents the floating part. Note that n means number of binary bits (base 2), not “number of decimals” as in a power of ten (base 10). Qx.18 is not the same as how we use 10^18 to store 1 ether. Qx.18 means “one” is 1 * 2^18 not 1 * 10^18.
By leftshift with n, an integer is converted to a Qx.n number.
By rightshifting with n, a Qx.n number is converted to an integer, that is the floating part ignored.

## Ch4: Sqrt Prices
UniswapV2 tracks the token reserves to derives spot price and liquidity from them. However, UniswapV3 tracks current price and liquidity to get the reserves. (The protocol stores the sqrt of price for gas saving, sqrtPriceX96 in slot0).
x96 stands for Q64.96 format. To recover the original sqrt price, just divide the value_Q64.96 / 2^96 .
- Maximum number stored by sqrtPriceX96 is 2^64 - 1. So, the maximum price can be (2^64)^2 = 2^128. It is roughly 10^38 in order of magnitude. Since token prices are always relative to another token, the price difference between two tokens in a pool can span up to 10^38 orders of magnitude.
- Minimum number stored by sqrtPriceX96 is 2^-96. So, the minimum price can be 2^-192. But most of the cases, that low numbers are not allowed.

## Ch5: Tick Limits
The minimum and maximum ticks indexes are hardcoded as MIN_TICK and MAX_TICK in the Uniswap v3 TickMath library. The tick may have values from -887.272 to 887.272 to satisfy the min and max prices. 

## Ch6: Tick Spacing and Fees
Each pool sets a tick spacing parameter to determine the possible tick for the pool. Crossing a tick in a swap incurs a gas cost, so ticks should be crossed as infrequently as possible during a typical trade. The more volatile the pair is, the wider the tick spaces are. But, this lessens the liquidity concentration.
Highly volatile assets tend to cause higher impermanent loss, while more stable assets tend to cause less impermanent loss. Thus, LPs will demand higher fees to compensate for impermanent loss when providing liquidity for highly volatile assets. 
The relationship between fee and tick spacing is contained in the mapping _feeAmountTickSpacing_ in the **UniswapV3Factory.sol** contract.
The initial relations are assigned in the factory constructor. To show this, the basis point must be determined at first. A basis point is 0.01% of 1% or simply 10^-6 as percentage. Then the fees and tick spaces can be mapped.
| Fee Basis | Tick Spacing |
| ------ | ------ |
| 500 | 10 |
| 3000 | 60 |
| 10000 | 200 |

## Ch7: Tick and Price
TichMath library provides two functions to convert tick and price one another, getSqrtRatioAtTick and getTickAtSqrtRatio. Chapter 7, 8, and 9 have a lot of math. Please refer to the course website if you are interested.

## Ch10: Real and Virtual Reserves
Real reserves are the actual amounts of tokens in a liquidity range. If a user add a liquidity in this range, at least reserve of one of the tokens must be greater than zero. Virtual reserves are the reserves like defined in UniswapV2. 
The real reserve of token X is the distance between the x-coordinate of the price and the x-coordinate of the upper tick. The same goes for token Y, but now it’s the distance in y to the lower tick. As the price (price of X in terms of Y) decreases, the amount of token Y depletes. On the contrary, while price increasing, the amount of token X gets depleting till the upper tick.

In Uniswap v3, a segment’s real reserves depend on the segment’s liquidity, the current price, and the segment’s boundaries—its upper and lower ticks.

Formulas for virtual reserves
```
where L = x*y and p = y/x

x = L / sqrt(p)
y = L * sqrt(p)
```

## Ch11: Real Reserves
Formulas for real reserves are very similar to virtual reserves. Consider the real reserves as the difference between current price and the boundry price, upper for x reserves and lower for y reserves.
```
where L = x*y and p = y/x

x = L / sqrt(p) - L / sqrt(p_u)
y = L * sqrt(p_l) - L * sqrt(p)
```

**!Note:** If the price is not in the liquidity range and it is lower than the lower tick, then all the reserves are only token X. x = L / sqrt(p_l) - L / sqrt(p _u). Similarly, if the current price is higher than upper tick, all the token X are swapped to token Y and there remains no reserves of token X.

### Delta Amounts
In UnswapV3, the protocol has to know whether the pool has enough reserves for a swap demand. Accordingly, it can fulfill the demand fully or partially. There are two functions for these calculations, getAmount0Delta for swapping token X and getAmount1Delta for swapping token Y.

## Ch12: Constant Product Formula
Basically in UniswapV2, the L is L = x*y. However the x and y in UniswapV3 are virtual reserves. So, if we want to represent the constant product equation using real reserves as a function of real reserves:
```
L^2 = (x_r + L /sqrt(p_u) * (y_r + L*sqrt(p_l)) 
```

## Ch13: Amounts of Real Reserves
- **getAmount0Delta:** Calculates the real reserves in X between two prices for a given liquidity.
- **getAmount1Delta:** Calculates the real reserves in Y between two prices for a given liquidity.