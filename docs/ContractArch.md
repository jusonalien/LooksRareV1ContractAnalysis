# LooksRare的主要部件：

- **Currency manager** (`CurrencyManager`) — 用来管理在系统里面允许用来做交易的ERC-20 Token白名单
- **Execution manager** (`ExecutionManager`) — . 交易订单撮合引擎
- **Transfer selector for NFT contracts** (`TransferSelectorNFT`) — NFT转账管理器
- **Royalty manager** (`RoyaltyFeeManager`) — 版权转让费管理器

## 合约代码：

[https://github.com/LooksRare/contracts-exchange-v1](https://github.com/LooksRare/contracts-exchange-v1)

在学习LooksRare的代码之前，如果不了解基础的交易术语，需要先掌握熟悉几个概念，不然会对一些函数的命名一头雾水

## Maker与Taker

- Maker就是挂单的人，即做市者，对单子里面的标的做出报价

- Taker就是吃单的人，如果有合适的单子就吃掉

记住，这里的Maker和Taker与买卖没关系，挂的单子可以是买单也可以是卖单

## Bid与Ask

- Bid是买

- Ask是卖
