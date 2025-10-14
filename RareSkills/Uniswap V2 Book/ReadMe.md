# Prerequisites
## Vault - ERC4626
It is a vault standard for ERC20 tokens. When you deposit a token A, then you get token S which represents your shares on the ERC4626 vault. When you turn your token S back, you take corresponding token As.
Each ERC4626 contract only supports one asset.

> Example:
You and 9 friends each deposite 10 DAI. The vault has 100 DAI in total. Let's say the vault earned 10 DAI more. The vault now has 110 DAI. When one of the provider wants to withdraw his shares, he gives his 10 shares back and gets 11 DAI in exchange. Now vault has 99 DAI and 9 providers, each has 11 DAI for their shares.

### Functions of ERC4626
- **asset()**: returns the address of the underlying token used for the Vault. If it is DAI, then the returned address is 0x6b175474e89094c44da98b954eedeac495271d0f
- **totalAssets()**: returns the number of ERC20 tokens owned by the ERC4626 vault.
- **deposit(uint256 assets, address receiver)**: you specify how many assets you want to put in, and the function will calculate how many shares to send to you. 
- **mint(uint256 shares, address receiver)**: you specify how many shares you want, and the function will calculate how much of the ERC20 asset to transfer from you. if you don’t have enough assets to transfer in to the contract, the transaction will revert.
- **previewDeposit()**: takes assets as an argument and returns the shares back without changing the state
- **previewMint()**: takes shares as an argument and returns the required assets back without changing the state.
- **convertToShares()**: takes assets as an argument and returns the amount of shares you will get back under ideal conditions (no slippage or fees).
- **withdraw()**: you specify how many assets you want to take from the contract, and the contract calculates how many of your shares to burn.
- **redeem()**: you specify how many shares you want to burn, and the contract calculates the amount of assets to give back.
- **previewWithdraw()**: view function to anticipate the withdraw
- **previewRedeem()**: view function to anticipate the redeem
- **convertToAssets()**: takes shares as an argument and gives you how many assets you will get back, not including fees and slippage.

### Slippage Problem
For automated market makers (AMMs), a large trade might use up the liquidity and cause the price to move substantially.
**OR**
A transaction gets frontrun or experiencing a sandwich attack.

__To defend for these__: the contract interacting with an ERC4626 should measure the amount of shares it received during a deposit (and assets during a withdraw) and revert if it does not receive the quantity expected within a certain slippage tolerance.

### Inflation Attack
> Example:
ERC4626 is independent to asset-to-share translation. Let's assume a vault with linear translation. Let's say there are 10000 assets and 100 shares. One can get 1 share buy depositing 100 assets. However, if one deposits 99 assets, the shares will round to zero and the provider gets 0 share. Noone can do this intentionally, but if there are 10,000 assets in the vault corresponding to 100 shares, and the attacker donates 20,000 assets, then one share is suddenly worth 300 assets instead of 100 assets. When the victim’s trade trades in assets to get back shares, they suddenly get a lot fewer shares — possibly zero.

There are three defenses:
1. Revert if the amount received is not within a slippage tolerance (described earlier)
2. The deployer should deposit enough assets into the pool such that doing this inflation attack would be too expensive
3. Add “virtual liquidity” to the vault so the pricing behaves as if the pool had been deployed with enough assets.

totalSupply(): total amount of shares
totalAssets(): the balance of the assets held by the vault

```
shares_received = assets_deposited * totalSupply() / totalAssets();
```

This formula is open to exploit. If an attacker frontruns and deposits assets which make totalAssets > assets_deposited * totalSupply(), the first depositor gets zero share due to the rounding of integer divison. If a virtual liquidity is added to totalSupply (i.e 1000), the attacker needs to deposit more assets (i.e 1000 times). This makes the attack nonprofitable.

## Flash Loans - ERC3156
Flash loans are loans between smart contracts that must be repaid in the same transaction.
Common use case: 
-> Borrower calls the lender contract's function like borrow().
-> borrow() calls a kind of callback function like onFlashLoan() in borrower contract.
-> onFlashLoan() executes some codes with its borrowed tokens and returns back the amount borrowed.
-> Once the borrower callback returns to lender's borrow(), function checks the balance whether loan paid back or not.
-> If not all code executions reverted.

**!Notice**: An EOA wallet cannot call a function to get the flash loan and then transfer the tokens back in a single transaction. Integration with a flash loan requires a separate smart contract.
**!Notice**: Flash loans do not need collateral, because a properly implemented flash loan function ensures the payback.

### Flash Loan Usecases
1. Arbitrage:  if Ether is trading for $1,200 in one pool and $1,300 in another DeFi application, it would be desirable to buy the Ether in the first pool and sell it in the second pool for a $100 profit. However, you need to money to buy the Ether in the first place. A flash loan is the ideal solution for it, as you don’t need $1,200 lying around. You can borrow $1,200 of Ether, sell it for $1,300, and pay back the $1,200 keeping a $100 profit for yourself (minus fees).
2. Refinancing Loans: If your stable coins loan had a 5% interest and you wanted to refinance with another lending smart contract at 4%, you would need to flash loans.
3. Liquidating Borrowers
4. Building a leverage loop in a single transaction

### Receiver Functions of ERC3156
The functions that the borrower needs to implement. 
- onFlashLoan()
    -  address initiator: It is the address that initiated the flash loan.
    -  address token: It is the address of the ERC20 token you are borrowing.
    -  uint256 fee
    -  bytes32 data

### Lender Functions of ERC3156
- flashLoan(): transfers the tokens to the receiver and transfer them back.

### Security Considerations
1. Access control and input validation for borrower: The borrowing smart contract must have the controls in place to only allow the flash lender contract to be the caller of onFlashLoan(). Otherwise, some actor other than the flash lender can call onFlashLoan() and cause unexpected behavior.
2. For the borrower, ensure only flash lender contract can call onFlashLoan
3. Using token.balanceOf(address(this)) can be manipulated

# UNISWAP V2
## AMM Architecture
UniswapV2 enables traders to swap one token for another in a trustless manner. Automated market makers (AMMs) are an alternative to an order book. An AMM holds two tokens (token X and token Y) in the pool (a smart contract). Anyone can withdraw token X in exchange for depositing token Y and vice versa. Let x and y be the amounts of token X and token Y. In a constant product AMM, multiplication of these amounts after each trade (swap) must be a constant like L. 
```
x*y = L^2
```
Assets are provided to the pool by liquidity providers, who receive so-called LP tokens to represent their share of the pool. Liquidity providers can only provide assets proportional to the current ratio of tokens in the pool. For example, if there are 100 token X and 200 token Y, the new liquidity provider must provide twice as many token Y as X. Liquidity provider balances are tracked in a manner similar to how ERC 4626 works. The difference between an AMM and ERC 4626 is that ERC 4626 only supports one asset but an AMM has two tokens.
The spot or marginal price is calculated my the current amount of tokens in the pool
```
P_x = y (amount of token Y that pool holds) / x (amount of token X that pool holds)
P_y = x (amount of token X that pool holds) / y (amount of token Y that pool holds)
```
The more one of the tokens you add into pool, the less the price of it you get. In other words, if you add token X to the pool, because its amount gets higher corresponding to the amount of token Y, its price goes down.

- AMMs do not have a bid-ask spread. Price always exists. 
- AMMs can be used for price oracle by other smart contracts.
- Every trade moves the price because it changes the amounts no matter how small the trade is. This results in a buy or sell order will generally encounter more slippage than in an order book model, and the mechanism of swapping invites sandwich attacks. A sandwich attack is that 1) Attacker’s first buy (front run): drives up price for victim 2) Victim’s buy: drive up price even further 3) Attacker’s sell: sell the first buy at a profit.
- Liquidity providers don’t have control over the price their assets are sold at. 
- UniswapV2 does not support limit orders.
- Liquidity providers for AMMs may suffer from impermanent loss.

> Impermanent Loss:
The scenario is that ETH price is 10$ at the beginnig and goes up to 1000$ later. Let's say Alice has 1 ETH and 10 USDC (20$ at the beginnig). When ETH price changed, she has 1010$ at the final (1 ETH + 10$) with the profit 990$. But, if she kept her assets in an AMM, she would have missed the profit. AMM would swap the ETH for USDC to satisfy the constant product equation after ETH price goes up. Initially the pool has 1 (ETH) * 10 (USDC) = 10. So, AMM has to swap ETH for USDC when the ETH reaches 1000$. By this way constant product is satisfied. Finally, the pool has 0.1 ETH and 100 USDC. This means Alice has 0.1 ETH and 100 USDC, 200 in total. Profit here is 180$. If she wouldn't have kept in AMM, she would have made 990-180 = 810$ more profit.

### Architecture
**UniswapV2Pair**: holds two ERC20 contracts. The traders can swap against, or liquidity providers can provide liquidity for. Every pair must be a different contract.
**UniswapV2Factory**: If a desired pair contract does not exist, factory creates it permissionlessly.
**UniswapV2Router02**: provides safer ways to interact with pair contracts. Also, it enables users to swap multi hop pairs.

UniswapV2 adopts a core-periphery design pattern. 

### Price Calculation
Consider an ETH / USDC trading pair with 100 ETH and 100 USDC in the Liquidity Pool. For simplicity, this assumes 1 ETH is 1 USDC. Although the spot price of 1 ETH is 1 USDC and vice versa, this does not mean we can trade 25 USDC for 25 ETH as this will not preserve the constant product formula.
**!Note**: The swap() function on Uniswap V2 requires you to pre-calculate the amount of tokens to swap from the pool (including 0.3% in trading fees).
As one token quantity increases (deposited into the AMM), the other should decrease (withdrawn from the AMM contract).
```
 x*y = L^2
    or
 y = L^2 / x
```
 L_before is the liquidity before the swap and L_after is the liquidity after swap. They must be equal.
```
L_before^2 = L_after^2
x_before * y_before = x_after * y_after
ETH_before * USDC_before = ETH_after * USDC_after
 ```
 
Uniswap V2 imposes a 0.3% AMM trading fee for every swap. When factoring in the fees, the constant product of the liquidity pool increases with each swap. This growth in the pool is the key incentives for liquidity providers. Only when liquidity providers withdraw their liquidity would the constant product of the pool decrease.
When you swap delta_ETH for delta_USDC, formula must satisfies the constant product equation
```
 ETH_before * USDC_before = ETH_after * USDC_after
 ETH_before * USDC_before = (ETH_before + delta_ETH) * (USDC_before - delta_USDC)
```
This means that for a pool starting with 100 ETH and 100 USDC (1 ETH = 1 USDC), you can't swap 25 ETH for 25 USDC. Because 100 * 100 = 10000 = L^2 and after swap this must be satisfied. So, for 25 ETH, the pool has 125 ETH and the USDC amount must be L^2 / 125 = 80 USDC, that is you can get 20 USDC for 25 ETH. This loss is also called _slippage loss_.
 
Uniswap V2 applies a 0.3% trading fee for every swap, but the fees only apply towards the token deposited into the AMM. For 25 ETH swap, fee is 0.075 ETH and remained depositing amount is 24.925 ETH. So, for the above example, when you extract the fees the amount of ETH in the pool will be 124.925 and corresponding USDC amount must be 80.048 that is you get 19.952 USDC back. 
 
### Swap Function
The function directly transfers out the amount of tokens that the trader requested in the function arguments without waiting the token in transferred. This is because providing flash loan property. 
> Callstack:
Borrowing Contract calls swap from pair contract
Pair contract transfers amount of tokens to the borrowing contract without any collateral
Pair contract calls a callback function in borrowing contract.
Callback function in borrowing contract uses the borrowed (transfered from pair contract) tokens and pay the tokens back.
If it doesn't, pair contract revert all previous executions.

**!Notice**: Flash swap is only for another contract, not for an EOA. Because an EOA cannot implement a callback function.

Swap function calculates the amount of deposited token (amount in) with the equation below:
```
currentContractBalanceX > previousContractBalanceX - _amountXOut
```
If the equation returns true, this means that the callee sent the amount of token X:
```
amountXIn = currentContractBalanceX - (_reserveX - amountXOut)
```
> Examples: 
Suppose our previous balance was 10, amountOut is zero, and currentBalance is 12. That means the user deposited 2 tokens. amountXIn will be 2.
Suppose our previous balance was 10, amountOut is 7, and currentBalance is 3. amountXIn will be 0.
Suppose our previous balance was 10, amountOut is 7, and currentBalance is 2. amountXIn will still be zero, not -1. It is true that the pool had a net loss of 8 tokens, but amountXIn cannot be negative.
Suppose our previous balance was 10, and amountOut is 6. If the currentBalance is 18, then the user “borrowed” 6 tokens but paid back 8 tokens.

**!Notice**: Uniswap V2 doesn’t prevent you from “paying too much” i.e. transferring in too many tokens in during the swap. 

### Burn Function
Liquidity is the balance of the pair contract. They are called LP tokens. Unless a user send a LP tokens to the pair contract, we basically assume that the contract has no LP tokens (zero LP token). If the contract had LP tokens, someone (an attacker) may burn it and get token0 and token1. So, A user must send the LP tokens to the contract when he wants to burn his shares.
```
uint256 liquidity = balanceOf(address(this));
```

### Mint Fee
In UniswapV2 1/6th of swap fees is separated to the protocol in the design (1/6th of 3% is 0.05%). But this feature never activated.
> Example:
At time t_1, the pool has 10 token0 and 10 token1. After several trades at time t_2, the pool now has 40 token0 and 40 token1. Liquidity l_1 = 10 and l_2 = 40. The pair contract collects fees from the l_2 - l_1. When mint() or burn() is called, Uniswap mints LP tokens to the protocol fee recipient. 

### TWAP
The price is a ratio in Uniswap. The ratio ignores the decimals of the pairs. So, you have to take the decimals in calculations into consideration. price of foo = reserve of foo / reserve of bar

Uniswap uses a fixed point number with 112 bits of precision on each side of the decimal, this takes up a total of 224 bits, and when packed with a 32 bit number, it uses up a single slot.

The motivation behind TWAP is: Measuring an instantaneous snapshot of assets in the pool leaves an opportunity for flash loan attacks. That is, someone can make a huge trade using a flash loan to cause a temporary dramatic shift in the price, then take advantage of another smart contract that uses this price to make decisions. So, spot price is too vulnarable  to any attack.

Using a time average price prevents the attackers to manipulate the price easily. An attacker has to manipulate the price for several consecutive blocks so that he can get a spiked price but this very costly. 
```
P_twap = (p1*t1 + p2*t2 + p3*t3 + ... + pn*tn) / (t1+t2+...+tn)
```

**!Notice**: Although the spot prices of token X and token Y are inversely proportional, their TWAP prices are not. This is easy to show some math operations.

In order to calculate TWAP for a certain amount of time, you should snapshot the cumulative prices. These prices are updated when the functions **swap, mint and burn** are called. But what if these functions are not called for a while and we don't have snapshots we need. The **sync** function is called to synchronize the reserves and balances. This helps updating and snapshoting the prices.

**!Notice**: PriceCumulativeLast is always increasing since the pool created. This inevitably causes an overflow. In that case, the previous reserve will be higher than the new reserve. When the oracle computes the change in price, they will get a negative value. However, this is desired for the protocol and this won’t matter due to the rules of modular arithmetic. 

> To make things simple let’s use an imaginary unsigned integers that overflow at 100. We snapshot the priceAccumulator at 80 and a few transactions/blocks later the priceAccumulator goes to 110, but it overflows to 10. We subtract 80 from 10, which gives -70. But the value is stored as an unsigned integer, so it gives -70 mod(100) which is 30. That’s the same result we would expect if it didn’t overflow (110-80=30).

## UniswapV2Library
It simplifies some interactions with pair contracts and is used heavily by the Router contracts. It contains eight functions that are not state-changing.
### getAmountOut() and getAmountIn()
It works with the equation below (ignoring fees for simplicity):
```
x*y = (x + delta_x) * (y - delta_y)
```
### pairFor()
It takes a factory address and token address, and returns the pair contract address using _create2_ method.

## Router
- **swapExactTokensForTokens**: its argument path includes input token address at first index and output token address at the last index. If the swap are hopping across pools, the path will specify [address(tokenIn), address(intermediateToken), …, address(tokenOut)]. This function tells the exact amount of input token to swap for output token. The user specifies exactly how much of the first token they are going to deposit and the minimum amount of the output token they will accept.

> Example:
Suppose we want to trade 25 token0 for 50 token1. If this is the exact price at the current state, this leaves no tolerance for the price changing before our transaction is confirmed, leading to a revert. So we instead specify the minimum out to be 49.5 token1, implicitly leaving a 1% slippage tolerance.

- **swapTokensForExactTokens**: This is the reverse function. You specifiy the amount of output token you want to get and the maximum amount of input token you are willing to deposit.

> In this case, we specify we want exactly 50 token1, but we are willing to trade up to 25.5 token0 to obtain in.

- **_swap**: external swap function takes amountOut parameters. Before calling the function the callee must send the tokens he wants to deposit. By this way, the swap function can calculate the amountIn by substracting reserve of the token from current balance of the token. The difference gives the amount in of deposited token. The function implements a for loop to traverse the path of multihop swap.
- **_addLiquidity**: adds liquidity by comparing the optimal amount of tokens can be deposited.

> Example:
Suppose the current pair balance is 100 token0 and 300 token1. We want to add 20 and 60 token0 and token1 respectively, but the pair ratio might change. So we instead approve the router for 21 token0 and 63 token1 while saying the minimum we want to deposit is 20 and 60. If the ratio shifts such that the optimal amount of token0 to deposit is 19.9, then the transaction reverts.

### Important Notes: 
- **deadline parameter**
When writing a smart contract that integrates with Uniswap, do not set the deadline to be block.timestamp or block.timestamp plus a constant. This can cause MEV and front-run attacks. Use deadline as an argument needs to be entered by the user.

- **Never set amountMin to zero or amountMax to type(uint).max**
Another very common mistake is to set the amountMin to zero or amountMax to a very high value. This destroys the protection against price slippage and sandwich attacks.
