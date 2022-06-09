# executionStrategies

## Looks的五种撮合策略

代码在contracts/executionStrategies目录下，说是有五种，**其实官网只公布了1、4、5的主网地址，其中策略1、5的手续费为2%，4的手续费为0**

### 1、**StrategyAnyItemFromCollectionForFixedPrice**

在Collection中任意符合报价的策略，举个例子，我要指定购买BAYC中随便一个NFT，只要价格符合74ETH的都可以，不挑剔！

**适用场景**：比较适合当手头的NFT为蓝筹的时候快速卖出，因为蓝筹都是抢手货～～

```solidity
// 这里只做了价格和时间的校验，为什么不去做collection的校验？
((makerBid.price == takerAsk.price) &&
                (makerBid.endTime >= block.timestamp) &&
                (makerBid.startTime <= block.timestamp))
```

### 2、**StrategyAnyItemInASetForFixedPrice**

适用场景：taker用一堆的tokenId生成了一个merkle树后（可以理解成一堆待售卖的token,从而构成一个Set），maker具备这个merkle的root，taker具备对应的tokenId以及对应的merkle proof ，验证通过后即可完成交易，如果对merkle proof的概念不理解，可以看这个：[https://ethbook.abyteahead.com/ch4/merkle.html](https://ethbook.abyteahead.com/ch4/merkle.html)

```solidity
// taker 给出证明树，taker 给出leaf 节点，以及root
bytes32 merkleRoot = abi.decode(makerBid.params, (bytes32));

// MerkleProof + indexInTree + tokenId
bytes32[] memory merkleProof = abi.decode(takerAsk.params, (bytes32[]));

// Compute the node
bytes32 node = keccak256(abi.encodePacked(takerAsk.tokenId));

// Return whether the order can be executed, the tokenId, and the amount to sell
return (
    (MerkleProof.verify(merkleProof, merkleRoot, node) &&
        (makerBid.price == takerAsk.price) &&
        (makerBid.endTime >= block.timestamp) &&
        (makerBid.startTime <= block.timestamp)),
    takerAsk.tokenId,
    makerBid.amount
);
```

### 3、StrategyDutchAuction

荷兰拍策略，价格随着时间的流逝不断的降低

```solidity
uint256 startPrice = abi.decode(makerAsk.params, (uint256));
uint256 endPrice = makerAsk.price;

uint256 startTime = makerAsk.startTime;
uint256 endTime = makerAsk.endTime;

// Underflow checks and auction length check
require(endTime >= (startTime + minimumAuctionLengthInSeconds), "Dutch Auction: Length must be longer");
require(startPrice > endPrice, "Dutch Auction: Start price must be greater than end price");

uint256 currentAuctionPrice = startPrice -
    (((startPrice - endPrice) * (block.timestamp - startTime)) / (endTime - startTime));

return (
    (startTime <= block.timestamp) &&
        (endTime >= block.timestamp) &&
        (takerBid.price >= currentAuctionPrice) &&
        (takerBid.tokenId == makerAsk.tokenId),
    makerAsk.tokenId,
    makerAsk.amount
);
```

### 4、StrategyPrivateSale

一对一交易，通过maker的order里面的params参数来指定taker进行交易

```solidity
address targetBuyer = abi.decode(makerAsk.params, (address));

return (
    ((targetBuyer == takerBid.taker) &&
        (makerAsk.price == takerBid.price) &&
        (makerAsk.tokenId == takerBid.tokenId) &&
        (makerAsk.startTime <= block.timestamp) &&
        (makerAsk.endTime >= block.timestamp)),
    makerAsk.tokenId,
    makerAsk.amount
);
```

### 5、StrategyStandardSaleForFixedPrice

普通标准交易，只要tokenId 和价格match就能进行撮合

```solidity
return (
            ((makerBid.price == takerAsk.price) &&
                (makerBid.tokenId == takerAsk.tokenId) &&
                (makerBid.startTime <= block.timestamp) &&
                (makerBid.endTime >= block.timestamp)),
            makerBid.tokenId,
            makerBid.amount
        );
```