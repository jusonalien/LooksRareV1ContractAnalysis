每当交易撮合发起Token转账单时候，需要用对应的NFT transfer来做转账，这样做的原因可能是不想直接在这执行token的transfer，防止出现安全问题导致资产损失,

可以看到，主网上对BAYC这类蓝筹项目都是有对应的NFT transfer的

![Untitled](/docs/pics/bayc-transfermanager.png)

所有的TransferManager由一个mapping变量进行管理

```solidity
// Map collection address to transfer manager address
mapping(address => address) public transferManagerSelectorForCollection;
```

## addCollectionTransferManager

管理员才能使用，给对应的Collection增加TransferManager

## removeCollectionTransferManager

管理员才能使用，删除对应COllection的TransferManager

## checkTransferManagerForToken

获取collection对应的TransferManager，可以看到即使管理员没有及时增加TransferManager也会去根据Token的类型返回默认的TransferManager