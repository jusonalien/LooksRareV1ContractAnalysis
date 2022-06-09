# CurrencyManager

用来管理在系统里面允许用来做交易的ERC-20 Token白名单

合约继承Ownable，使得白名单的**增加，删除**的功能只允许管理员才能操作

- addCurrency 增加一种Token
- removeCurrency 移除一种Token
- isCurrencyWhitelisted 检查某Token是否在白名单内
- viewCountWhitelistedCurrencies 获取token的总量数字
- viewWhitelistedCurrencies 通过下标来获取对应的token信息 cursor + size