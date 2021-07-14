---
title: DeFi发展史：Uniswap V1 源码阅读和分析
date: 2021-07-13 06:47:23
mathjax: true
tags:
- DeFi
- Uniswap
---

Uniswap V1版本在2018年11月启动（创始人启动协议的[Twitter](https://twitter.com/haydenzadams/status/1058376395108376577)）。Uniswap V1从本质上来说是一个去中心化的自动交易所，提供ETH和ERC-20代币之间进行兑换的功能。Uniswap V1协议的出现为基于Ethereum的DeFi生态发展奠定了基础。根据官网的[文档](https://uniswap.org/docs/v1/)所描述，Uniswap V1在设计之初注重去中心化、去监管化（censorship resistance）和安全性，是一个开源的公共产品（public good），不收取交易费用。

## 背景：ERC-20代币

下面首先简要介绍一下Ethereum中的代币。代币（Token）在Ethereum生态中可以代表一切可以量化的东西，比如货币（如对标美元的DAI代币和代表ETH本身的代币WETH）、股份（如向资金池投资所返回的资金池代币）等等。ERC-20代币标准是2015年在[EIP-20](https://eips.ethereum.org/EIPS/eip-20)中提出，作为Ethereum智能合约体系中代币实现的统一标准。代币的本身也是一种智能合约，定义了一系列行为，并记录了每个地址所拥有的代币数量。ERC-20代币接口的实现如下：

```solidity
interface ERC20Interface {
    // 任何ERC-20代币必须实现如下接口
    // 查询代币的总供给量
    function totalSupply() public view returns (uint);
    // 查询某个地址所拥有的代币量
    function balanceOf(address tokenOwner) public view returns (uint balance);
    // 允许spender（从调用者这里）取走一定量的代币
    function approve(address spender, uint tokens) public returns (bool success);
    // 查询tokenOwner允许spender取走的代币数量
    function allowance(address tokenOwner, address spender) public view returns (uint remaining);
    // 将tokens数量的代币转移给地址to
    function transfer(address to, uint tokens) public returns (bool success);
    // 从地址from向地址to转账tokens数量的代币
    function transferFrom(address from, address to, uint tokens) public returns (bool success);
    
    // 以下功能是可选的
    function name() external view returns (string);
    function symbol() external view returns (string);
    function decimals() external view returns (string);

    // 转账过程可能触发的事件
    event Transfer(address indexed from, address indexed to, uint tokens);
    event Approval(address indexed tokenOwner, address indexed spender, uint tokens);
}
```

## Uniswap V1体系概览

Uniswap V1本质上是一个自动的交易所，能够自动地和用户交互并兑换ETH和ERC-20代币，兑换比例的确定（即*价格*）采用恒定乘积自动做市系统（constant-product automated market maker），也就是说，交易所资金池内的ETH和ERC-20代币数量的乘积总体是恒定的。和传统交易所的OrderBook撮合机制不同，在自动做市机制中每个用户的交易对象（交易对手方）都是交易所本身，而交易所通过公式确定兑换比例。不同交易所（如去中心化的自动化交易所和Binance等传统的交易所）之间的价格一致性由套利机制进行实现：如果交易所之间的价格存在差异，那么就有投机者进行投机交易（如低买高卖）将价格差异缩小至套利机会不存在（即交易费用大于套利利润）。

V1版本的Uniswap协议采用[Vyper](https://github.com/vyperlang/vyper)语言进行开发（而非当前Ethereum合约开发主流的Solidity语言）。Vyper是一种语法和Python非常接近的语言。Ethereum智能合约发展初期，由于Solidity语法相对独特且发展较为缓慢，Vyper曾经是Ethereum合约开发的主流语言之一，随后才被快速发展且受到官方支持的Solidity取代。

Uniswap V1的体系相对来说比较简单，分为两个部分：

* Exchange，用于进行ETH和ERC-20代币之间的兑换。
* Factory，用于创建和记录所有的Exchange，也用于查询代币对应的Exchange。

## Factory的实现

Factory的实现非常简单，代码如下：

```python
contract Exchange():
    def setup(token_addr: address): modifying

NewExchange: event({token: indexed(address), exchange: indexed(address)})

exchangeTemplate: public(address)
tokenCount: public(uint256)
token_to_exchange: address[address]
exchange_to_token: address[address]
id_to_token: address[uint256]

@public
def initializeFactory(template: address):
    assert self.exchangeTemplate == ZERO_ADDRESS
    assert template != ZERO_ADDRESS
    self.exchangeTemplate = template

@public
def createExchange(token: address) -> address:
    assert token != ZERO_ADDRESS
    assert self.exchangeTemplate != ZERO_ADDRESS
    assert self.token_to_exchange[token] == ZERO_ADDRESS
    exchange: address = create_with_code_of(self.exchangeTemplate)
    Exchange(exchange).setup(token)
    self.token_to_exchange[token] = exchange
    self.exchange_to_token[exchange] = token
    token_id: uint256 = self.tokenCount + 1
    self.tokenCount = token_id
    self.id_to_token[token_id] = token
    log.NewExchange(token, exchange)
    return exchange

@public
@constant
def getExchange(token: address) -> address:
    return self.token_to_exchange[token]

@public
@constant
def getToken(exchange: address) -> address:
    return self.exchange_to_token[exchange]

@public
@constant
def getTokenWithId(token_id: uint256) -> address:
    return self.id_to_token[token_id]
```

* `initializeFactory`只在创建的时候被调用，一旦设置了`template`参数后就无法更改，确保了用于创建Exchange的代码模板不会被修改。`template`是链上部署的合约，用于作为后续创建的Exchange的模板。
* `createExchange`用于从模板创建一个Exchange。在做一些必要的校验之后，代码调用内置函数`create_with_code_of`拷贝`exchangeTemplate`所指示的地址中的代码创建一个新的合约并返回其地址。随后调用新创建的Exchange的setup函数设置代币地址，并将新创建的Exchange记录在合约内。注意到在做验证的过程中，函数约束每一个代币只能对应一个Exchange，这是为了约束某个代币的所有流动性都划分在一个池子中，增加池子中对应的存储量，降低交易的滑点。

## Exchange的实现

从Factory的代码可以看出，Uniswap的核心功能在Exchange中进行实现。Exchange的实现略有复杂，首先给出所有功能的接口，接下来按照每个接口的功能介绍其实现：

```solidity
// 只保留了Exchange的核心功能接口
interface UniswapExchangeInterface {
    // 流动性
    function addLiquidity(uint256 min_liquidity, uint256 max_tokens, uint256 deadline) external payable returns (uint256);
    function removeLiquidity(uint256 amount, uint256 min_eth, uint256 min_tokens, uint256 deadline) external returns (uint256, uint256);
    // 价格查询
    function getEthToTokenInputPrice(uint256 eth_sold) external view returns (uint256 tokens_bought);
    function getEthToTokenOutputPrice(uint256 tokens_bought) external view returns (uint256 eth_sold);
    function getTokenToEthInputPrice(uint256 tokens_sold) external view returns (uint256 eth_bought);
    function getTokenToEthOutputPrice(uint256 eth_bought) external view returns (uint256 tokens_sold);
    // 提供ETH以兑换代币
    function ethToTokenSwapInput(uint256 min_tokens, uint256 deadline) external payable returns (uint256  tokens_bought);
    function ethToTokenTransferInput(uint256 min_tokens, uint256 deadline, address recipient) external payable returns (uint256  tokens_bought);
    function ethToTokenSwapOutput(uint256 tokens_bought, uint256 deadline) external payable returns (uint256  eth_sold);
    function ethToTokenTransferOutput(uint256 tokens_bought, uint256 deadline, address recipient) external payable returns (uint256  eth_sold);
    // 提供代币以兑换ETH
    function tokenToEthSwapInput(uint256 tokens_sold, uint256 min_eth, uint256 deadline) external returns (uint256  eth_bought);
    function tokenToEthTransferInput(uint256 tokens_sold, uint256 min_eth, uint256 deadline, address recipient) external returns (uint256  eth_bought);
    function tokenToEthSwapOutput(uint256 eth_bought, uint256 max_tokens, uint256 deadline) external returns (uint256  tokens_sold);
    function tokenToEthTransferOutput(uint256 eth_bought, uint256 max_tokens, uint256 deadline, address recipient) external returns (uint256  tokens_sold);
    // 代币之间的互换
    function tokenToTokenSwapInput(uint256 tokens_sold, uint256 min_tokens_bought, uint256 min_eth_bought, uint256 deadline, address token_addr) external returns (uint256  tokens_bought);
    function tokenToTokenTransferInput(uint256 tokens_sold, uint256 min_tokens_bought, uint256 min_eth_bought, uint256 deadline, address recipient, address token_addr) external returns (uint256  tokens_bought);
    function tokenToTokenSwapOutput(uint256 tokens_bought, uint256 max_tokens_sold, uint256 max_eth_sold, uint256 deadline, address token_addr) external returns (uint256  tokens_sold);
    function tokenToTokenTransferOutput(uint256 tokens_bought, uint256 max_tokens_sold, uint256 max_eth_sold, uint256 deadline, address recipient, address token_addr) external returns (uint256  tokens_sold);
}
```

### 添加/取回流动性

用户可以调用`addLiquidity`和`removeLiquidity`向资金池中添加和取回流动性。

`addLiquidity(min_liquidity: uint256, max_tokens: uint256, deadline: timestamp) -> uint256`

向资金池添加流动性。Uniswap V1中添加流动性的过程简述如下：
1. 用户调用`addLiquidity`函数并发送一定量的ETH。
2. Uniswap Exchange要求用户按照当前资金池中的ETH和代币的比例添加流动性。Uniswap通过比较发送的ETH量和池子中的ETH数量，计算用户需要发送的代币数量$Token_{Deposited}$，并将该数量的代币从用户（交易的Sender）转给自己。
3. 为了证明用户确实提供了流动性及用户流动性所占的份额，Uniswap Exchange将向用户发放LP（Liquidity Pool）代币，其数量为$Amount_{LPToken}$。

$$Token_{Deposited}=Token_{Pool}*\frac{ETH_{Deposited}}{ETH_{Pool}}$$

$$
Amount_{LPToken}=TotalSupply_{LPToken}*\frac{ETH_{Deposited}}{ETH_{Pool}}
$$

由于Uniswap去中心化的特性，添加流动性的交易发出时和确认时流动性池的兑换比例（或者说价格）可能不同。为了避免这个问题给用户造成的损失，`addLiquidity`函数提供了三个参数进行控制：

* `min_liquidity`：用户期望的LP代币数量。如果最终产生的LP代币数量过少，则交易会回滚避免损失。
* `max_tokens`：用户想要提供的最大代币量。如果计算得出的代币数量大于这个参数，代表用户不愿意提供更多代币，交易也会回滚。
* `deadline`：时限。如果交易确认的区块时间大于deadline，也会回滚。

```python
# 第一部分：total_liquidity > 0
@public
@payable
def addLiquidity(min_liquidity: uint256, max_tokens: uint256, deadline: timestamp) -> uint256:
    assert deadline > block.timestamp and (max_tokens > 0 and msg.value > 0)
    # total_liquidity = totalSupply of LP token
    total_liquidity: uint256 = self.totalSupply

    if total_liquidity > 0:
        assert min_liquidity > 0
        # eth & token reserve in the pool
        eth_reserve: uint256(wei) = self.balance - msg.value
        token_reserve: uint256 = self.token.balanceOf(self)
        # the amount of token user should also provide 
        token_amount: uint256 = msg.value * token_reserve / eth_reserve + 1
        # minted amount of LP token
        liquidity_minted: uint256 = msg.value * total_liquidity / eth_reserve
        assert max_tokens >= token_amount and liquidity_minted >= min_liquidity
        # record LP token balance & totalSupply
        self.balances[msg.sender] += liquidity_minted
        self.totalSupply = total_liquidity + liquidity_minted
        # transfer tokens from user to Exchange
        assert self.token.transferFrom(msg.sender, self, token_amount)

        log.AddLiquidity(msg.sender, msg.value, token_amount)
        log.Transfer(ZERO_ADDRESS, msg.sender, liquidity_minted)
        return liquidity_minted
```

`removeLiquidity(amount: uint256, min_eth: uint256(wei), min_tokens: uint256, deadline: timestamp) -> (uint256(wei), uint256)`

```python
def removeLiquidity(amount: uint256, min_eth: uint256(wei), min_tokens: uint256, deadline: timestamp) -> (uint256(wei), uint256):
    assert (amount > 0 and deadline > block.timestamp) and (min_eth > 0 and min_tokens > 0)
    # total_liquidity = totalSupply of LP token
    total_liquidity: uint256 = self.totalSupply
    assert total_liquidity > 0
    token_reserve: uint256 = self.token.balanceOf(self)
    # calculate returned eth & token amount
    eth_amount: uint256(wei) = amount * self.balance / total_liquidity
    token_amount: uint256 = amount * token_reserve / total_liquidity
    # check
    assert eth_amount >= min_eth and token_amount >= min_tokens
    # update status
    self.balances[msg.sender] -= amount
    self.totalSupply = total_liquidity - amount
    # send eth & token back
    send(msg.sender, eth_amount)
    assert self.token.transfer(msg.sender, token_amount)
    # log event
    log.RemoveLiquidity(msg.sender, eth_amount, token_amount)
    log.Transfer(msg.sender, ZERO_ADDRESS, amount)
    return eth_amount, token_amount
```

### 价格查询

下面首先介绍Uniswap V1的价格机制。每个Exchange（或者说一个池子）中有且只有两种资产：ETH和代币，池子中两个资产存量（Reserve）的比率构成了价格，用户可以在ETH和代币以及代币和代币之间自由兑换。因此，用户有两种指定价格的方式：精确指定换出（Output）值，并限定最大的输入值（Input）；或者精确指定换入（Input）值，并设置最小的输出值（Output）。

因此，Uniswap V1在实现中首先实现了两个私有函数作为定价体系：`getInputPrice`和`getOutputPrice`。

`getInputPrice(input_amount: uint256, input_reserve: uint256, output_reserve: uint256)`

在确定池子中输入单位和输出单位的存量时，精确的输入数量能换出的输出数量。不难看出，该函数实现了这样一个公式：

$$
Output=Reserve_{Output} * \frac{Input * 997}{Input * 997 + Reserve_{Input} * 1000}
$$

输入单位的$0.3\%$作为交易费用，剩下的输入进入池子。因此分母为更新后池子中输入单位的存量，分子为除去交易费用后的输入，分式表达的是*输入进入池子后*输入在池子中所占的份额。该分式乘以输出池子的存量即为输入对应的输出量。

```python
def getInputPrice(input_amount: uint256, input_reserve: uint256, output_reserve: uint256) -> uint256:
    assert input_reserve > 0 and output_reserve > 0
    input_amount_with_fee: uint256 = input_amount * 997
    numerator: uint256 = input_amount_with_fee * output_reserve
    denominator: uint256 = (input_reserve * 1000) + input_amount_with_fee
    return numerator / denominator
```

`getOutputPrice(output_amount: uint256, input_reserve: uint256, output_reserve: uint256)`

在确定池子中输入单位和输出单位的存量时，精确的输出数量能换出的输入数量。不难看出，该函数实现的公式是由上面的式子变换而来：

$$
Input=Reserve_{Input} * \frac{Output * 1000}{997 * (Reserve_{Output} - Output)} + 1
$$

```python
def getOutputPrice(output_amount: uint256, input_reserve: uint256, output_reserve: uint256) -> uint256:
    assert input_reserve > 0 and output_reserve > 0
    numerator: uint256 = input_reserve * output_amount * 1000
    denominator: uint256 = (output_reserve - output_amount) * 997
    return numerator / denominator + 1
```

Uniswap提供价格查询的四个函数：

* `getEthToTokenInputPrice`
* `getEthToTokenOutputPrice`
* `getTokenToEthInputPrice`
* `getTokenToEthOutputPrice`

均在这两个函数的基础上进行实现。以`getEthToTokenInputPrice`为例：

```python
def getEthToTokenInputPrice(eth_sold: uint256(wei)) -> uint256:
    assert eth_sold > 0
    token_reserve: uint256 = self.token.balanceOf(self)
    return self.getInputPrice(as_unitless_number(eth_sold), as_unitless_number(self.balance), token_reserve)
```

可以看出，在实现时向`getInputPrice`中传入的存量参数`{input,output}_reserve`分别是Exchange本身的ETH数量和Exchange在代币合约中记录的代币存量。

### ETH和代币间的互换

有了价格计算函数，ETH和代币间的互换就变得非常直观。首先来看内部实现的四个函数：

通过精确的ETH输入量（`eth_sold`）计算价格并交换代币。通过`getInputPrice`计算输出的代币数量。同样包含了`min_tokens`最小代币输出量和`deadline`的时间限制。

```python
@private
def ethToTokenInput(eth_sold: uint256(wei), min_tokens: uint256, deadline: timestamp, buyer: address, recipient: address) -> uint256:
    # check
    assert deadline >= block.timestamp and (eth_sold > 0 and min_tokens > 0)
    # calculate output token
    token_reserve: uint256 = self.token.balanceOf(self)
    tokens_bought: uint256 = self.getInputPrice(as_unitless_number(eth_sold), as_unitless_number(self.balance - eth_sold), token_reserve)
    assert tokens_bought >= min_tokens
    # transfer token to recipient
    assert self.token.transfer(recipient, tokens_bought)
    log.TokenPurchase(buyer, eth_sold, tokens_bought)
    return tokens_bought
```

通过精确的代币输出量（`tokens_bought`）计算价格并交换代币。通过`getOutputPrice`计算输入的ETH数量，并可能在ETH需求量小于用户发送量时产生Refund。同样包含了`min_tokens`最小代币输出量和`deadline`的时间限制。

```python
@private
def ethToTokenOutput(tokens_bought: uint256, max_eth: uint256(wei), deadline: timestamp, buyer: address, recipient: address) -> uint256(wei):
    # check
    assert deadline >= block.timestamp and (tokens_bought > 0 and max_eth > 0)
    # calculate input ETH
    token_reserve: uint256 = self.token.balanceOf(self)
    eth_sold: uint256 = self.getOutputPrice(tokens_bought, as_unitless_number(self.balance - max_eth), token_reserve)
    # may have refund, also check (revert) if eth_sold > max_eth
    eth_refund: uint256(wei) = max_eth - as_wei_value(eth_sold, 'wei')
    if eth_refund > 0:
        send(buyer, eth_refund)
    # transfer token
    assert self.token.transfer(recipient, tokens_bought)
    log.TokenPurchase(buyer, as_wei_value(eth_sold, 'wei'), tokens_bought)
    return as_wei_value(eth_sold, 'wei')
```

`tokenToEthInput`和`tokenToEthOutput`在实现上与上面两个函数基本一致。

在交易机制上，Uniswap V1实现了两种交易方式：Swap和Transfer。两者的唯一差别在于，Swap调用的接收者固定为交易发送者（即`msg.sender`），而Transfer调用可以额外指定一个接收者。通过如下函数对可以清晰地看出：

```python
@public
@payable
def ethToTokenSwapInput(min_tokens: uint256, deadline: timestamp) -> uint256:
    # 'receipient' is msg.sender
    return self.ethToTokenInput(msg.value, min_tokens, deadline, msg.sender, msg.sender)

@public
@payable
def ethToTokenTransferInput(min_tokens: uint256, deadline: timestamp, recipient: address) -> uint256:
    assert recipient != self and recipient != ZERO_ADDRESS
    # 'receipient' is specified as parameter
    return self.ethToTokenInput(msg.value, min_tokens, deadline, msg.sender, recipient)
```

参考：

* [Uniswap V1 白皮书](https://hackmd.io/C-DvwDSfSxuh-Gd4WKE_ig#ERC20-%E2%87%84-ERC20-Trades)

Uniswap V1在Ethereum MainNet中的合约地址：

* [Factory](https://etherscan.io/address/0xc0a47dfe034b400b47bdad5fecda2621de6c4d95)
* [Exchange Template](https://etherscan.io/address/0x2157a7894439191e520825fe9399ab8655e0f708)
* [交易所案例：ETH-DAI](https://etherscan.io/address/0x2a1530c4c41db0b0b2bb646cb5eb1a67b7158667)