** BNB测试网股票锚定代币项目（假设是：AAPLx） 模块说明文档**。


# 技术栈
  * Solidity ^0.8.20
  * Hardhat/remix
  * OpenZeppelin Contracts Upgradeable
  * OpenZeppelin Hardhat Upgrades 插件
  * Chainlink Price Feeds
---

# aaplx_token 合约模块说明

```
contracts/
├── AAPLxToken.sol       // 主逻辑合约
├── ProxyAdmin.sol       // 代理管理员
├── AAPLxProxy.sol       // ERC1967代理合约
├── PriceFeed.sol        // Chainlink价格获取
├── USDTVault.sol        // USDT托管与白名单提取
└── AccessControl.sol    // 权限与黑名单管理
```

---

## AAPLxToken.sol

* 定义 AAPLx 的主 ERC20 逻辑合约，实现可升级模式（ERC1967 / UUPS）。
* 功能设计：

  * 标准 ERC20 功能：`transfer`, `approve`, `transferFrom`, `balanceOf`, `totalSupply`。
  * 铸造与销毁函数：`mint`, `burn`。
    * ***销毁逻辑***销毁逻辑中:用户下单售出AAPLX,交易程序操作并返回交易状态,根据交易状态的内容(时间/价格/数量/成交状态),对应销毁AAPLX,并把USDT转给用户.
  * 下单函数：

    * `place_order(
        address usdtAddress, // 使用代币合约地址
        uint256 amount, // 使用代币数量
        uint256 price, // 订单价格
        uint256 qty, // 订单数量 
        string calldata code, // 标的代码
        TrdSide trd_side, // 交易方向
        OrderType order_type // 订单类型
    )`：用户下单。

   
 *  `emit PlaceOrder(
            msg.sender,
            usdtAddress,
            amount,
            price,
            qty,
            code,
            trd_side,
            order_type
        );
`。打印下单日志
  * 集成黑名单校验：转账、兑换、赎回前检查。
  * 集成可暂停功能：暂停后所有用户交互函数不可调用。
  * 初始化器支持升级安全部署。

---

## AAPLxProxy.sol

* 部署时指定初始逻辑合约地址与初始化数据。
* 转发所有调用到当前逻辑合约。
* 使用 OpenZeppelin ERC1967 设计，确保安全升级：

  * 可调用 `upgradeTo(newImplementation)` 实现合约升级。
  * 管理员地址记录在 EIP-1967 的标准存储槽。

---

## ProxyAdmin.sol

* 管理代理合约的升级逻辑。
* 功能：

  * 调用代理的 `upgradeTo()`。
  * 变更逻辑合约实现。
  * 仅 ProxyAdmin 的 Owner 可以升级。

---

## PriceFeed.sol

* 封装 Chainlink 的 Aggregator 接口。
* 功能：

  * 设置 Aggregator 地址。
  * 提供 `getLatestPrice()` 返回最新 AAPL/USD 价格。
  * 可由任何用户调用（只读）。
* 部署时配置 Chainlink Aggregator 地址。

---

## USDTVault.sol

* 合约内托管用户兑换支付的 USDT。
* 功能：

  * 记录白名单地址（可提取USDT）。
  * 只有白名单地址可调用 `withdrawUSDT(uint256 amount)`。
  * 拥有者可调用 `addToWhitelist(address)`、`removeFromWhitelist(address)` 管理权限。
  * 存储所有待赎回USDT余额。

---

## AccessControl.sol

* 实现管理员/操作员权限。
* 功能：

  * onlyOwner 权限修饰符，用于关键操作。
  * 管理黑名单地址列表。

    * `addToBlacklist(address)`。
    * `removeFromBlacklist(address)`。
  * 转账、兑换、赎回均需检查是否在黑名单。
  * 暂停控制：

    * `pause()`：暂停所有用户功能。
    * `unpause()`：恢复功能。
  * 暂停状态下，仅管理员能调用管理函数。

---

## 兑换与赎回流程图

```mermaid
graph LR
    subgraph User
        A[用户USDT] --> B[place_order()]
        B --> C[AAPLxToken合约]
        C --> D[铸造AAPLx]
        D --> E[用户AAPLx余额增加]
    end

    subgraph Reverse
        E --> F[place_order()]
        F --> C
        C --> G[销毁AAPLx]
        G --> H[USDT转给用户]
    end
```

---

## 黑名单/白名单管理流程图

```mermaid
graph TD
    subgraph Owner
        A1[addToWhitelist()] --> B1[USDTVault]
        A2[removeFromWhitelist()] --> B1

        A3[addToBlacklist()] --> B2[AccessControl]
        A4[removeFromBlacklist()] --> B2
    end

    subgraph User
        C1[withdrawUSDT()] --> B1
        C2[transfer()] --> B2
        C3[exchange()/redeem()] --> B2
    end
```

---

## ⚙️ 部署流程

1. 部署逻辑合约 AAPLxToken（UUPSUpgradeable）。
2. 部署 ProxyAdmin。
3. 部署 AAPLxProxy，指定 ProxyAdmin 与 AAPLxToken。
4. 部署 PriceFeed，配置 Chainlink Aggregator。
5. 部署 USDTVault。
6. 配置 AccessControl（添加 Owner、Operator、初始白名单、黑名单）。
7. 连接前端/脚本测试兑换和赎回流程。

---
