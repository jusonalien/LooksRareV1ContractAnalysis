# LooksRareExchange

这个合约将CurrencyManager、RoyaltyFeeManager，executionStrategies，TransferSelectorNFT都关联了起来，组成一套完整的LooksRare交易所的核心功能，可以看到，开发者在设计的时候就考虑很周详，里面的几个模块功能都提供了对应的升级函数，防止相关核心模块出现漏洞的时候可以做出及时修复

这个合约涉及到的函数比较多，就按照合约结构里面函数的顺序来一一分析

## 两个重要的成员变量

```solidity
// 记录用户的nonce值
mapping(address => uint256) public userMinOrderNonce;
// 记录order是否失效（已执行or被取消）
mapping(address => mapping(uint256 => bool)) private _isUserOrderNonceExecutedOrCancelled;
```

## cancelAllOrdersForSender

取消用户的所有订单，合约调用方才能取消自己的订单

```solidity
require(minNonce > userMinOrderNonce[msg.sender], "Cancel: Order nonce lower than current");
require(minNonce < userMinOrderNonce[msg.sender] + 500000, "Cancel: Cannot cancel more orders");
userMinOrderNonce[msg.sender] = minNonce;

emit CancelAllOrders(msg.sender, minNonce);
```

## cancelMultipleMakerOrders

取消用户的Maker订单

```solidity
require(orderNonces.length > 0, "Cancel: Cannot be empty");

for (uint256 i = 0; i < orderNonces.length; i++) {
    require(orderNonces[i] >= userMinOrderNonce[msg.sender], "Cancel: Order nonce lower than current");
    _isUserOrderNonceExecutedOrCancelled[msg.sender][orderNonces[i]] = true;
}

emit CancelMultipleOrders(msg.sender, orderNonces);
```

## matchBidWithTakerAsk

如果拿到一个NFT找到合适价格的订单快速卖出就用这个函数

可以看到，

1. 验证订单的真实性
2. 交易策略是由maker在下订单的时候定的，taker只能遵循maker的策略
3. 更新 _isUserOrderNonceExecutedOrCancelled 
4. 转移交易对象NFT
5. 转移款项

```solidity
require((!makerBid.isOrderAsk) && (takerAsk.isOrderAsk), "Order: Wrong sides");
require(msg.sender == takerAsk.taker, "Order: Taker must be the sender");

// Check the maker bid order
bytes32 bidHash = makerBid.hash();
_validateOrder(makerBid, bidHash);

// 检查maker给出的策略是否满足
(bool isExecutionValid, uint256 tokenId, uint256 amount) = IExecutionStrategy(makerBid.strategy)
    .canExecuteTakerAsk(takerAsk, makerBid);

require(isExecutionValid, "Strategy: Execution invalid");

// Update maker bid order status to true (prevents replay)
_isUserOrderNonceExecutedOrCancelled[makerBid.signer][makerBid.nonce] = true;

// 先转交易对象，NFT
// Execution part 1/2
_transferNonFungibleToken(makerBid.collection, msg.sender, makerBid.signer, tokenId, amount);

// 再转支付款
// Execution part 2/2
_transferFeesAndFunds(
    makerBid.strategy,
    makerBid.collection,
    tokenId,
    makerBid.currency,
    makerBid.signer,
    takerAsk.taker,
    takerAsk.price,
    takerAsk.minPercentageToAsk
);
```

## matchAskWithTakerBid

想买入某个NFT的时候如果价格合适就用这个接口，逻辑跟matchBidWithTakerAsk是大同小异的，不做重复描述

## matchAskWithTakerBidUsingETHAndWETH

逻辑和matchAskWithTakerBid的相同，只是是用ETH来完成交易

## 更新模块合约功能

逻辑比较简单

updateCurrencyManager

updateExecutionManager

updateProtocolFeeRecipient

updateRoyaltyFeeManager

updateTransferSelectorNFT

isUserOrderNonceExecutedOrCancelled

## _transferFeesAndFunds

转账功能：手续费+款项

手续费包含了looksRare协议本身的手续费+ 版权转让费用 royalty fee，其中royalty的管理遵循EIP-2981

协议手续费：由具体的撮合策略决定

版权转让费用：由具体的collection决定，有些会有有些没有。要看对应的ERC721 token是否有实现对应的EIP-2981的接口了。

```solidity
uint256 finalSellerAmount = amount;

// 1. Protocol fee
{
    uint256 protocolFeeAmount = _calculateProtocolFee(strategy, amount);

    // Check if the protocol fee is different than 0 for this strategy
    if ((protocolFeeRecipient != address(0)) && (protocolFeeAmount != 0)) {
        IERC20(currency).safeTransferFrom(from, protocolFeeRecipient, protocolFeeAmount);
        finalSellerAmount -= protocolFeeAmount;
    }
}

// 2. Royalty fee
{
    (address royaltyFeeRecipient, uint256 royaltyFeeAmount) = royaltyFeeManager
        .calculateRoyaltyFeeAndGetRecipient(collection, tokenId, amount);

    // Check if there is a royalty fee and that it is different to 0
    if ((royaltyFeeRecipient != address(0)) && (royaltyFeeAmount != 0)) {
        IERC20(currency).safeTransferFrom(from, royaltyFeeRecipient, royaltyFeeAmount);
        finalSellerAmount -= royaltyFeeAmount;

        emit RoyaltyPayment(collection, tokenId, royaltyFeeRecipient, currency, royaltyFeeAmount);
    }
}

require((finalSellerAmount * 10000) >= (minPercentageToAsk * amount), "Fees: Higher than expected");

// 3. Transfer final amount (post-fees) to seller
{
    IERC20(currency).safeTransferFrom(from, to, finalSellerAmount);
}
```

## _transferFeesAndFundsWithWETH

逻辑跟_transferFeesAndFunds一样，只是直接用WETH完成转账

## _transferNonFungibleToken

```solidity
// Retrieve the transfer manager address
address transferManager = transferSelectorNFT.checkTransferManagerForToken(collection);

// If no transfer manager found, it returns address(0)
require(transferManager != address(0), "Transfer: No NFT transfer manager available");

// If one is found, transfer the token
ITransferManagerNFT(transferManager).transferNonFungibleToken(collection, from, to, tokenId, amount);
```

## _calculateProtocolFee

协议的手续费由具体的撮合策略决定

```solidity
uint256 protocolFee = IExecutionStrategy(executionStrategy).viewProtocolFee();
return (protocolFee * amount) / 10000;
```

## _validateOrder

利用零知识证明的方法方法去验证order是否是真的由maker地址生成的