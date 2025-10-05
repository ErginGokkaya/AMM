# Constant Product AMM

L^2 = x*y : This equation determines the valid amounts of token A and token B.

For example: Token A, Token B parities may be (100,400) (200,200) (400,100) are valid. But (300,300) or (100,200) doesn't meet the equation. So they are invalid.

- Let's consider an AMM with starting amount of 200 token A and 200 token B. (L^2 = 200 * 200)
If someone deposit 200 token B, then L^2 = 200 * 200 = newAmountOfTokenA * 400. So newAmountOfTokenA must be 100.
So the depositor gets 100 (200 - 100) token A.
If someone deposit 100 token A, then L^2 = 200*200 = 300 * newAmountOfTokenB. So newAmountOfTokenB must be 133.33.
So the depositor gets 66,67 (200 - 133.33) token B.

- L is the liqudity. The bigger the L is, the bigger the swap returns
## case 1 (L^2 = 200 * 200)
If someone deposit 200 token B, then L^2 = 200 * 200 = newAmountOfTokenA * 400. So newAmountOfTokenA must be 100.
So the depositor gets 100 (200 - 100) token A.
## case 2 (L^2 = 400 * 400)
If someone deposit 200 token B, then L^2 = 400 * 400 = newAmountOfTokenA * 600. So newAmountOfTokenA must be 266.67.
So the depositor gets 133.33 (400 - 266.67) token A.

# Types of Contracts
## Factory contract
It deploys the pair contracts.
## Pair contract
It is a contract holding a pair of ERC-20 tokens.
Enaples the users to add or remove liqudity.
Allows traders to swap ERC-20 tokens locked inside the pair contract.
Example pairs: ETH/USDT, DAI/ETH, MKR/DAI
Users can directly call the pair contracts to add or remove liquidity and swap.
This direct access usualy error prone. User may accidentally lock his tokens inside the contract. That's why the router contracts are defined.
## Router contract
It is an intermediate contarct between the user and the pair contracts.
It can interact with a single pair contract or multiple contract for chain swap

- User can directly access to factory contract so that it creates a pair contract which doesn't exist. Or he may tell the the router contract to call the factory contract to create a pair contract.

# Examples
## Swap Example (without fee)
L^2 = x*y --> this is the curve where the amount of tokens move

At the beginning, let's say the amount of tokens are x0 and y0. (They meet the eq. L^2 = x0 * y0)

Alice wants to swap dx for dy. (Alice deposits token X in exchange for token Y)
So now amount of X : x0 + dx and amount of Y : y0 - dy
This amounts must also meet the eq. L^2 = (x0 + dx) * (y0 - dy)
After some math (no trick, simple math :) ), dy can be find with the eq. dy = (y0*dx) / (x0 + dx)

## Swap Example (with fee)
0 <= f <= 1 : fee rate. Token in will be charged on fee

swap fee: f*dx

When the user pay the fee, the amount of deposit is dx * (1-f)
New dy eq is --> dy = ( dx * (1-f) * y0 ) / ( x0 + dx * (1-f))

## Router Example 
Consider a swap for WETH to DAI (Single Hop Swap)  
Let the user call the swap function inside the router contract. For us to swap ERC20 to ERC20, we should call swapExactTokensForTokens function on Router.
Then user transfer WETH to Pair Contract.
Once the WETHs transfered, Router calls the swap function on Pair Contract. And checks the balance of the contract.
Then the Pair contract calculates the corresponding DAI. Then sent them to the user.

!! Homework: search for what the stack is too deep error 

## Swap in Pair Contract

Formula is this:
balance is the amount after deposit
reserve is the amount before deposit
amountOut is the input parameter which represents the amount to get

amountIn = balance > reserve - amountOut ? balance - ( reserve - amountOut ) : 0
This is to check whether the balance increased. if the balance increased, this means that that token is deposited.

## Spot Price (Current Price) of The Tokens
Basically, spot price can be found by the tangent line of the tokens' amount point on the L-curve
Let's say the initial amounts are x0 and y0. Then dx is deposited. New point is (x0+dx, y0-dy).

execution price = -dy / dx : expression for how much dy we get for each dx deposit.

This is also the slope of the swap line.
Besides the swap formula without fee tells us that dy = dx * y0 / (x0 + dx)
while dx -> 0 (meaning that the trade amount getting smaller and smaller), formula turns to be:

-dy/dx = y0/x0

## Slippage
It is the difference between the actual price and the expected price.
Example:
Before swap -> you have 2000 DAI to be deposited to get 1 ETH (Which the price is AMM calculated )
ETH Price on AMM => 2000 DAI / 1 ETH => 2000
You accept it and sent you DAIs
After swap -> you get 0.99 ETH in exchenge for 2000 DAI
let's assume the fee = 0

ETH Price now => 2000 DAI / 0.99 ETH => 2020 

- Causes of Slippage
1. Market Movement

# Creating Pair Pool
1. Using Router Contract
2. Using Factory Contract

Both calls createPair function with the arguments of the addresses of Token A and Token B.
If the pair doesn't already exist, the function gets pair contact creation code.
A creation code is runtime code plus constructor arguments. (A runtime code is contract byte code which is run when a transaction occurs.)
Then this creation code is used to get the address of the pair contract.
To do this, the function uses the create2 assembly code. The inputs are creation code and a salt.
The salt here is hash of the addresses of token A and token B.
create2 gives us the pair contract address before the related contract deployed.
To make sure that the create2 address is only dependent on the token addresses, the constructor of UniswapV2PairContract takes no argument. Consider, if constructor takes argument, create2 address would also be dependent on that argument's value.

# Pool Shares
- A UniswapV2 user can be a liquidity provider by providing a pair of tokens.
Then these tokens are used to swap. The sawps collet fees, and then the liquidity provider can claim his fee shares.
The shares minted to the liquidity provider is calculated with the formula of s:

s = ((L1 - L0) / L0 ) * T

s:  shares minted to the provider
T:  amount of total current shares
L0: amount of tokens before the provider deposits
L1: amount of tokens after the provider deposits

L1    T + s
-- = -------  ==> the idea is simple. the increase on amount of tokens is proportional to the increase on total shares.
L0      T


- When a user(provider) wants to burn his shares, he gets b ( equals to L0 - L1 ) token back

b = ( s / T ) * L0

s:  shares burned by the provider
T:  amount of total current shares
L0: amount of tokens before the provider withdraws
L1: amount of tokens after the provider withdraws

  T - s     L1
 ------- = ----  ==> the idea is similar. decrease on total shares is proportional to decrease on amount of tokens. 
    T       L0

# Adding Liquidity 
(? Smart Contract Programmer says like that.. But I don't agree.. Because when the curve moves, it is no longer a constant product curve.. Check this deeply)
(! I think I get the idea.. when we add liquidity, this doesn't mean that the product of tokens is constant.. the product is constant when only we swap the tokens.. Don't confuse the swap with adding or removing liquidity.. Adding or removing liquidty changes the L, of course)
You may find L = f(x,y) = sqrt(x*y). L0 = sqrt(x0 * y0) and L1 = sqrt((x0+dx) * (y0+dy))

When we add liquidity to a constant product AMM, the curve moves towards upper right of the graph.
This movement determines the amounts of token pairs will be deposited.
Consider the spot price section, there are a pair of tokens which satisfies the a point on the L-curve.
After deposit, L-curve moves to upper right. But the amounts of tokens have to satisfy the same point.

  y0     dy
 ---- = ----
  x0     dx

- Merging Pool Shares to Liquidity

s = (( L1 - L0 ) / L0) * T    ...remember the eq above

(( L1 - L0 ) / L0) = dx / x0 = dy / y0  ==> s = dx * T / x0 = dy * T / y0


- AMM price (spot price) is always moving. So addLiquidity function can't accept the exact amount of tokens.
So it takes amountDesired and amountMin of tokens. The _to is the address which is the receiver of corresponding pool shares. 

!! Homework: Search for vault inflation attack

# Removing Liquidity
User calls removeLiquidity function on UniswapV2Router contract.

The idea is the same. Removing liquidity results in a lower L curve and the AMM price (spot price) remains the same.

The amount of burnt liquidity is:

L0 - L1 = (s / T) * L0

s: shares to burn
T: total shares
L0: liquidity before
L1: liquidity after
dx : the amount of token A to be withdrawn
dy : the amount of token B to be withdrawn
L0 - L1 : the decrement in liquidity

  L0 - L1     dx     dy
 --------- = ---- = ----
    L0        x0     y0

# Flash Swap

A smart contract can borrow and then repaid in the same transaction.
A user start a flash swap to contract. Then this contract calls swap function on UniswapV2PairContract.
The pair contract borrows the tokens without getting any tokens from the initiating contract.
Then the pair contract calls uniswapV2Call function from the initiating contract.
The initiating contract uses the borrowed tokens in any operations like arbitrage.
Then in the same transaction, it transfers the borrowed tokens + fee to the pair contract.

dx0 = amount of token borrowed
dx1 = repay of the borrow (dx0 + fee)
x0  = reserve before

flash swap fee equation: x0 - dx0 + 0.997 dx1 >= x0

fee = (3/997) * dx0

# TWAP (Time Weighet Average Pricing)
Problem: determine the price of a token. in web2 it is easy to get the price using an API call. However, in web3 it is not that easy. You need to use price oracle contract. You may have an idea to use the uniswap spot price as an actual price. But it has some dangers. First of all, it is easy to manipulate.
Consider a lending contract, this protocol allows you to borrow DAI when you lock your ETH as collateral.
Let's say max amount of borrow is 80% of collateral in DAI. Then say the contract gets the spot price of ETH/DAI uniswap pair contract as the price of DAI.
User lock 1 ETH to lending protocol(contract). Then protocol gets the spot price of DAI/WETH pair contract and it is 2000 DAIWETH. So max borrow is 2000 * 80/100 = 1600 DAI. Let's say that user borrows 1000 DAI.
Think of a hacker wants to exploit the lending protocol. Let's say this hacker has 1e7 DAI and 100 WETH. First he wants the WETH price expensive by buying WETH from the pair. He swaps 10e6 DAI to get 832 WETH. This makes WETH price spiked up. Then he borrow DAI by locking 100 WETH to lending protocol as collateral. Then he borrows the max amount of DAI. Lending protocol gets the spot price of DAI/WETH from the pair as 71819 DAIWETH. This means that the hacker can borrow 5745599 DAI. Then he put 832 ETH to pair contract to swap for DAI. Then he gets 9989964 DAI back. He spent 10e6 DAI and 100 WETH and profited around 15e6 DAI. He gets almost 5e6 DAI by just manipulating the uniswap pair spot price.

- TWAP as a solution
P_i : price of token X in terms of token Y between time t_i <= t < t_(i+1)
dt_i = t_(i+1) - t_i  (this is not a constant. it may vary by the index i. this means t_3 != t_4. this is because the index i is dependent of block.timestamp which is related to the transaction. As you know, transactions happens at irregular time)
t : current time
P : current price

TWAP from t_0 to t_n:
          n-1
          ---    dt_i
P_twap =  \     ------- * P_i
          /     t_n-t_0
          ---
          i=0

! Off-Topic: Never Forget Solidity Does Not Support Decimals

# Arbitrage
Let's say there are two AMM which have same pairs.
Let's say both have DAI/WETH pair, and one of the is cheaper than the other. It means that there is a risk free arbitrage opportunity.
Even if the user has no DAI to utilize this opportunity, he can borrow by using a flash swap or something and do arbitrage.

!! Homework: search for MEV- front run

## Maximum profit for arbitrage
Assume there are 2 uniswapv2 AMM, AMM A and AMM B. Also assume that P_B > P_A (constant product of AMM B is greater than AMM A for the same token pair).
!! I confused a lot again.. I am gonna check this later. I need to be sure about what the spot price, AMM price, prices on L-curve, price changes when an amount add or remove.
What I mean that, he gives us a two AMM for same pair. I guess all the prices on a curve are the same. If the prices are different, I would assume that the L-curves are different. I mean higher the price is, further to upper right the L-curve moves. 
If the prices are same, I assume that the curves are the same. Although they have same price (x*y = k) values, the combination of the amounts of the tokens may change for different points on the curve. Then this difference results in different prices of tokens which creates arbitrage opportunity. 

# LINK: https://updraft.cyfrin.io/courses/uniswap-v2