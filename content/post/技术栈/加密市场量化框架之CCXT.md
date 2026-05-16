---
title: "加密市场量化框架之CCXT"
description: "ccxt是加密货币量化领域最受欢迎的量化框架，本文对其API及设计理念做了相关讲解"
date: 2026-05-15
lastmod: 2026-05-15
weight: 4
categories:
    - Tutorial
    - Tech Stack
    - Quant
tags:
    - 教程
    - 技术栈
    - 量化投资
    - ccxt
---

# 加密市场量化框架之CCXT

- 作者：山财小蒋
- 联系方式：2018036661@qq.com
- 创作不易，转载请注明出处，欢迎批评探讨

---

- **ccxt是一个用于加密货币量化交易的python库，也是最受欢迎的加密量化框架，它分为ccxt和ccxt pro两个版本，其中xxct是基于http的轮询机制，而ccxt pro则是基于websocket的事件驱动机制，现在均已开源；**

- **ccxt库的实际理念是“统一”，加密货币市场极具开源性，有几百家交易所，但ccxt库综合了这几百家交易所并提供了一套优美的API模版**

## 两种连接机制：

- 下面先来讲讲我刚刚提到的两种机制，部分读者可能对这里不是很了解：

### http与https：

- **http超文本传输协议**，**一问一答、短连接**
- 客户端发请求 → 服务器立刻回数据 → **连接立刻断开**
- 只能**客户端主动问**，服务器**不能主动发消息**
- 每次请求都要重新握手、带请求头，开销大

- https就是 **HTTP + 加密（SSL/TLS）**
- 底层逻辑和 HTTP 一模一样，还是**一问一答短连接**
- 区别：传输内容加密，防被窃听、篡改，现在网站、交易所API基本都是 HTTPS

### websocket：

- **长连接、双向通信、事件驱动协议**
- 先通过 HTTP 握手升级，之后**一条连接一直连着不断开**
- 客户端可以发消息，**服务器也能主动推送消息**给客户端
- 没有每次都握手的开销，延迟极低，适合实时行情、订单推送

- 一句话总结：
- **HTTP/HTTPS**：一次性问答，用完就断；只能客户端主动问
- **WebSocket**：长久通话，一直在线；双方可以随时互发消息

---

### 轮询（CCXT 普通版 REST 就是轮询）

- **我每隔一段时间主动去问一次服务器**
- 比如每 1 秒、3 秒调用一次 `fetchTicker`、`fetchOrderBook`
- 不管行情有没有变化，你都在反复请求
- 本质：**定时主动拉数据**

- 特点
- 基于 **HTTP/HTTPS**
- 有**固定延迟**：你1秒轮询一次，行情变了最多要等1秒你才知道
- 请求量大，容易**被交易所限流**
- 浪费带宽：大部分时候行情没变化，你还在反复请求
- 适合：低频策略、拿历史数据、非实时需求

---
### 事件驱动：

- **我建立长连接等着，服务器有变化主动通知我**
- 连上 WebSocket 后就挂在那
- 盘口变了、价格变了、成交了 → **交易所主动推消息过来**
- 程序收到事件再处理，不用你主动反复问
- 本质：**有变化才触发，被动接收推送**

- 特点
- 基于 **WebSocket** 长连接
- **延迟极低**，毫秒级感知行情变化
- 请求极少，只建一次连接，几乎不限流
- 带宽占用极小
- 自带自动重连、心跳，不用自己写定时任务
- 适合：套利、网格、做市、实时监控仓位订单

## 实例化交易所：

### 创建交易所：

- 以下是实例化交易所的两种方式，以及交易所ID的两种动态写法：

```python
import ccxt
print (ccxt.exchanges) #只读属性，打印所有可用的交易所信息以及当前ccxt库版本信息

# 1. 基础方式：创建币安实例
exchange = ccxt.binance()  # 通过exchanges查询到的币安默认id（就是'binance'）

# 2. 为同一个币安交易所创建多个账户实例
binance1 = ccxt.binance({'id': 'binance1'})
binance2 = ccxt.binance({'id': 'binance2'})

# 3. 动态方式1：eval（不推荐，仅作演示，环境不安全）
exchange_id = 'binance'
binance = eval('ccxt.%s()' % exchange_id)

# 4. 动态方式2：getattr（推荐）
binance = getattr(ccxt, 'binance')()

## 来自变量 id（标准写法，带API Key）
exchange_id = 'binance'
exchange_class = getattr(ccxt, exchange_id)
exchange = exchange_class({
    'apiKey': 'YOUR_API_KEY',
    'secret': 'YOUR_SECRET',
    'enableRateLimit': True,  # 强烈建议开启，防止被限流
})
```

### 覆盖交易所属性：
- CCXT 里每个交易所类，都有一套默认属性（比如请求频率限制、默认请求头、签名方式、时间同步策略等），覆盖交易所属性就是在创建交易所实例时，用我们自己写的参数，替换掉库里面的默认值。
目的是：
- 调整程序的请求频率，避免被交易所限流 / 封号
- 自定义请求头，绕过部分限制或满足特殊接口要求
- 调整交易所特定的配置（比如时间同步、合约模式等）

```python
# Python
exchange = ccxt.binance({
    'rateLimit': 10000,   # 指定请求间隔（ms），rateLimit 是所有交易所通用的属性
    'headers': {
        'YOUR_CUSTOM_HTTP_HEADER': 'YOUR_CUSTOM_VALUE',
    },  # 自定义请求头，模拟特定的浏览器、客户端，绕过部分限制；添加额外的身份验证头；
    'options': {
        'adjustForTimeDifference': True,  # 交易所特定选项
    }
})
exchange.options['adjustForTimeDifference'] = False

```

- **adjustForTimeDifference**是币安的 API 签名，对时间戳要求非常严格（误差超过 ±1000ms 就会报错 Timestamp for this request is outside of the recvWindow）
- 设为 True 时，CCXT 会自动计算本地和服务器的时间差，并在请求时自动修正时间戳，解决时间不同步的问题
- 这是币安特定的选项，不是所有交易所都有，所以注释写了交易所特定选项

- **exchange.options['adjustForTimeDifference'] = False**实例创建完成后，动态修改 options 里的配置
这行代码把 adjustForTimeDifference 从 True 改成了 False
- 作用：
- 关闭自动时间同步，使用本地时间直接请求，适合我们手动校准了服务器时间，或者不想让 CCXT 自动修正时间戳的场景


### Testnets和Sandbox环境：
- 一些交易所还提供用于测试目的的单独API，使开发人员可以免费交易虚拟货币并测试他们的想法。这些API被称为 “testnets”、“sandboxes” 或者 “staging environments”（具有虚拟测试资产的环境），与 “mainnets” 和 “production environments”（具有真实资产的环境）相对应。即测试网，大多数情况下，沙盒API是生产API的克隆，所以它们基本上是相同的API，除了与交易所服务器的URL不同。

- CCXT统一了这一方面，并允许用户切换到交易所的沙盒（如果底层交易所支持）。 要切换到沙盒模式，用户必须在创建交易所后 立即调用 exchange.setSandboxMode (true) 或 exchange.set_sandbox_mode(true) 之前，而且不能在之前有其他API的调用！

```python
exchange = ccxt.binance(config)
exchange.set_sandbox_mode(True)  # enable sandbox mode
```

- 在创建交易所后立即调用 **exchange.setSandboxMode (true) / exchange.set_sandbox_mode (True)**，在其他API调用之前

- 沙盒API和主网（真实）API不一样，是不互通的，要获取API keys以访问沙盒，用户必须在相关交易所的沙盒网站上注册并创建沙盒 API 密钥对

- 沙盒密钥与生产密钥不能互换，主网的 API Key，在沙盒里用不了，反之亦然

---

## 交易所结构化：

### 覆盖交易所属性：

- 每个交易所都有一组属性和方法，大多数情况下，你可以通过将参数的关联数组传递给交易所构造函数来覆盖这些属性和方法。你也可以创建一个子类并覆盖所有内容。

- 这是一个带有示例值的通用交易所属性概览：

```python
{
    'id':   'exchange'                   // 小写字符串交易所 id
    'name': 'Exchange'                   // 可读的字符串
    'countries': [ 'US', 'CN', 'EU' ],   // ISO 国家代码数组
    'urls': {
        'api': 'https://api.example.com/data',  // API 基本 URL 的字符串或字典
        'www': 'https://www.example.com'        // 网站 URL 的字符串
        'doc': 'https://docs.example.com/api',  // URL 字符串或URL数组
    },
    'version':         'v1',             // 以数字结尾的字符串
    'api':             { ... },          // API 端点的字典
    'has': {                             // 交易所功能
        'CORS': false,
        'cancelOrder': true,
        'createDepositAddress': false,
        'createOrder': true,
        'fetchBalance': true,
        'fetchCanceledOrders': false,
        'fetchClosedOrder': false,
        'fetchClosedOrders': false,
        'fetchCurrencies': false,
        'fetchDepositAddress': false,
        'fetchMarkets': true,
        'fetchMyTrades': false,
        'fetchOHLCV': false,
        'fetchOpenOrder': false,
        'fetchOpenOrders': false,
        'fetchOrder': false,
        'fetchOrderBook': true,
        'fetchOrders': false,
        'fetchStatus': 'emulated',
        'fetchTicker': true,
        'fetchTickers': false,
        'fetchBidsAsks': false,
        'fetchTrades': true,
        'withdraw': false,
    },
    'timeframes': {                      // 如果exchange.has['fetchOHLCV'] !== true，则为空
        '1m': '1minute',
        '1h': '1hour',
        '1d': '1day',
        '1M': '1month',
        '1y': '1year',
    },
    'timeout':           10000,          // 毫秒数
    'rateLimit':         2000,           // 毫秒数
    'userAgent':        'ccxt/1.1.1 ...' // 字符串，HTTP User-Agent 标头
    'verbose':           false,          // 布尔值，输出错误详情
    'markets':          { ... }          // 交易对的字典
    'symbols':          [ ... ]          // 按字符串排序的交易对列表
    'currencies':       { ... }          // 按币种代码的货币字典
    'markets_by_id':    { ... },         // 根据 id 的 array of dictionaries (markets) 的字典
    'currencies_by_id': { ... },         // 根据 id 的字典的字典 (markets) 的字典
    'apiKey':   '92560ffae9b8a0421...',  // 公共的 apiKey 字符串 (ASCII, hex, Base64, ...)
    'secret':   '9aHjPmW+EtRRKN/Oi...'   // 私有的 secret key 字符串
    'password': '6kszf4aci8r',           // password 字符串
    'uid':      '123456',                // user id 字符串
    'options':          { ... },         // 交易所特定的选项
    // ... 这里还有其他属性 ...
}
```

- 以下上述代码中是每个基本交易所属性的详细描述：

- **id**：每个交易所都有一个默认的 id。这个 id 不用于任何用途，它是一个字符串文字，用于用户级别的交易所实例标识目的。您可以对同一交易所有多个链接，并通过 id 进行区分。默认 id 全部小写，并与交易所名称对应。

- **name**：这是一个包含易于理解的交易所名称的字符串文字。

- **countries**：一个包含交易所所在地的二字母 ISO 国家代码的字符串文字数组。

- **urls['api']**：用于 API 调用的单个字符串文字基本 URL，或者一个关联数组，其中包含私有和公共 API 的单独 URL。

- **urls['www']**：主要的 HTTP 网站 URL。

- **urls['doc']**：指向交易所 API 原始文档的单个字符串 URL 链接，或者指向文档的链接数组。

- **version**：包含当前交易所 API 版本标识符的字符串文字。ccxt 库在每个请求上附加此版本字符串到 API 基本 URL 上。除非您正在实现一个新的交易所 API，否则不需要修改它。版本标识符通常是以字母 “v” 开头的数值字符串，如 v1.1。除非您正在实现自己的新加密交易所类，否则不要覆盖它。

- **api**：一个包含密码交换所公开的所有 API 端点定义的关联数组。API 定义用于 ccxt 自动构建每个可用端点的可调用实例方法。- has: 这是一个交易所能力的关联数组（例如 fetchTickers、fetchOHLCV 或 CORS）。

- **timeframes**: 一个关联数组，包含交易所的 fetchOHLCV 方法支持的时间框架。只有在 has['fetchOHLCV'] 属性为 true 时才会填充该数组。

- **timeout**: 请求-响应往返的超时时间，以毫秒为单位（默认超时时间为 10000 ms = 10 秒）。如果在此时间内未收到响应，库将抛出 RequestTimeout 异常。你可以使用默认的超时值，也可以将其设置为一个合理的值。毫无超时的悬挂是不可行的，当然。一般情况下，你不必覆盖此选项。软件

- **rateLimit**: 请求限制速率，以毫秒为单位。指定两个连续的对同一交易所的HTTP请求之间的最小延迟。内置的速率限制器默认是启用的，可以通过将 enableRateLimit 属性设置为 false 来关闭它。

- **enableRateLimit**: 一个布尔 (true/false) 值，启用内置的速率限制器，限制连续的请求。默认情况下，此设置为 true（启用）。用户需要实现自己的速率限制或者保持内置的速率限制器启用，以避免被交易所封禁。

- **userAgent**: 一个对象，用于设置 HTTP User-Agent 头。ccxt 库默认会设置它的 User-Agent。有些交易所可能不喜欢它。如果你无法从交易所收到回复并希望关闭 User-Agent 或使用默认值，请将此值设置为 false、undefined 或空字符串。userAgent 的值可以被下面的 HTTP headers 属性覆盖。

- **headers**: 一个关联数组，包含 HTTP 头和它们的值。默认值为空 {}。所有头将被添加到所有请求的前面。如果在 headers 中设置了 User-Agent 头，它将覆盖上面 userAgent 属性设置的任何值。

- **verbose**: 一个布尔标志，指示是否将 HTTP 请求记录到 stdout（verbose 标志默认为 false）。Python 用户有一种替代方法，可以使用标准的 Pythonic 记录器进行调试日志记录，只需在代码开头添加以下两行：
```python
import logging
logging.basicConfig(level=logging.DEBUG)
```
- **markets**: 一个关联数组，通过常见的交易对或符号进行索引的市场。在访问此属性之前，应该先加载市场。在交易所实例上调用 loadMarkets() / load_markets() 方法之前，市场是不可用的。

- **symbols**: 一个非关联数组（列表），包含交易所提供的可用符号，并按字母顺序排序。这些是 markets 属性的键。符号是从市场加载和重新加载的。此属性为所有市场键提供了方便的简写形式。- currencies: 一个关联数组（字典），按代码（通常是3或4个字母）列出了可用于交换的货币。货币会从市场中加载和重新加载。

- **markets_by_id**: 一个关联数组，按交易所特定的id索引的市场数组列表。通常只包含一个元素，除非有多个具有相同marketId的市场。在访问此属性之前，应该先加载市场。

- **apiKey**: 这是您的公共API密钥字符串。大多数交易所都需要设置API密钥。

- **secret**: 您的私密API密钥字符串字面量。大多数交易所也需要此密钥和apiKey一起使用。脚本语言

- **password**: 一个包含您的密码/短语的字符串字面量。某些交易所在交易时需要此参数，但大多数交易所不需要。

- **uid**: 您账户的唯一id。这可以是一个字符串字面量或数字。某些交易所在交易时也需要这个id，但大多数交易所不需要。

- **requiredCredentials**: 一个统一的关联字典，显示发出私有API调用到底层交易所所需的API凭证（交易所可能需要特定的一组密钥）。

- **options**: 一个特定于交易所的关联字典，包含底层交易所接受并在CCXT中支持的特殊键和选项。

- **precisionMode**: 交易所十进制精度计算模式，请阅读更多关于精确度和限制的信息

- 有关代理 - proxyUrl，httpUrl，httpsUrl，socksProxy：特定代理的URL。请在代理部分详细了解。请参考覆盖交换属性部分。

### 交换元数据：
- **has**：一个包含交换能力标志的关联数组，可以告诉我们哪些功能能用，哪些不能用，即这个交易所支持什么功能，包括以下内容：

```python
'has': {

    'CORS': false,  // 是否启用了跨域资源共享（来自浏览器） 

    // 统一的方法可用性标志（可以为true、false或'emulated'）：

    'cancelOrder': true,
    'createDepositAddress': false,
    'createOrder': true,
    'fetchBalance': true,
    'fetchCanceledOrders': false,
    'fetchClosedOrder': false,
    'fetchClosedOrders': false,
    'fetchCurrencies': false,
    'fetchDepositAddress': false,
    'fetchMarkets': true,
    'fetchMyTrades': false,
    'fetchOHLCV': false,
    'fetchOpenOrder': false,
    'fetchOpenOrders': false,
    'fetchOrder': false,
    'fetchOrderBook': true,
    'fetchOrders': false,
    'fetchStatus': 'emulated',
    'fetchTicker': true,
    'fetchTickers': false,
    'fetchBidsAsks': false,
    'fetchTrades': true,
    'withdraw': false,
    ...
}

```

- 每个标志表示该方法的可用性，具体含义如下：

- undefined / None / null的值表示ccxt目前未实现该方法（ccxt尚未统一该方法或交换API原生不支持该方法）

- boolean值false表示交换API原生不支持该端点

- boolean值true表示该端点在交换API中原生可用，并在ccxt库中统一

- 字符串'emulated'表示交换API原生不支持该端点，但在ccxt库中通过其他可用的真实方法进行了重构（尽可能）

- 有关所有交换和它们支持的方法的完整列表，请参阅以下示例：**https://github.com/ccxt/ccxt/blob/master/examples/js/exchange-capabilities.js##** 速率限制

- 交易所通常会设置所谓的速率限制。交易所会记住并跟踪你的用户凭据和IP地址，如果你对API的查询太频繁，他们将不允许你继续查询。他们通过平衡负载和控制流量拥堵来保护API服务器免受(D)DoS攻击和滥用。

- 警告：请保持在速率限制范围内，以避免被禁止！

- 大多数交易所允许每秒最多1~2个请求。如果你对您的请求过于激进，交易所可能会暂时限制您访问其API或封禁你一段时间。

- exchange.rateLimit属性默认设置为安全值，但这不是最优的。一些交易所对不同的终端点可能有不同的速率限制。用户根据特定应用目的来调整rateLimit是用户的责任。

- CCXT库内置了一个实验性的速率限制器，它会在后台为用户透明地进行必要的节流控制。警告：用户负责至少某种类型的速率限制：可以通过实现自定义算法或使用内置速率限制器来实现。

使用.enableRateLimit属性来开启/关闭内置速率限制器，示例如下：

```python
# 启用实例化交换时的内置速率限制
exchange = ccxt.bitfinex({
    # 'enableRateLimit': True,  # 默认启用
})

# 或者在实例化后以后随时打开或关闭内置速率限制
exchange.enableRateLimit = True  # 启用
exchange.enableRateLimit = False  # 禁用
```

- 如果您的调用达到速率限制或出现nonce错误，ccxt库将抛出InvalidNonce异常，或者在某些情况下可能抛出以下类型之一：

- **DDoSProtection**

- **ExchangeNotAvailable**

- **ExchangeError**

- **InvalidNonce**

- 通常，稍后的重试足以处理此问题

### 关于速率限制器的注意事项:

- 速率限制器是交换实例的一个属性，换句话说，每个交换实例都有自己的速率限制器，不知道其他实例的存在。在许多情况下，用户应该在整个程序中重复使用同一个交换实例。不要使用同一IP地址上具有相同API密钥对的多个相同交换实例。

```python
// 不要这样做！

const binance1 = new ccxt.binance ({ enableRateLimit: true })
const binance2 = new ccxt.binance ({ enableRateLimit: true })
const binance3 = new ccxt.binance ({ enableRateLimit: true })

while (true) {
    const result = await Promise.all ([
        binance1.fetchOrderBook ('BTC/USDT'),
        binance2.fetchOrderBook ('ETH/USDT'),
        binance3.fetchOrderBook ('ETH/BTC'),
    ])
    console.log (result)
}
```

- 尽量重复使用交换实例，如下所示:

```python
// 做这个代替：

const binance = new ccxt.binance ({ enableRateLimit: true })

while (true) {
    const result = await Promise.all ([
        binance.fetchOrderBook ('BTC/USDT'),
        binance.fetchOrderBook ('ETH/USDT'),
        binance.fetchOrderBook ('ETH/BTC'),
    ])
    console.log (result)
}
```

- 由于速率限制器属于交换实例，销毁交换实例也会销毁速率限制器。在使用速率限制时，最常见的陷阱之一就是反复创建和销毁交换实例。如果在程序中你反复创建和销毁交换实例（比如，在多次调用的函数内），那么你就会不断地重置速率限制器，最终会破坏速率限制。如果你每次重新创建交换实例而不是重复使用它，CCXT 将会尝试重新加载市场。因此，你将不断地强制加载市场，如 Loading Markets 部分所述。滥用市场接口最终也会破坏速率限制器。

```python
// DO NOT DO THIS!

async function tick () {
    const exchange = new ccxt.binance ({ enableRateLimit: true })
    const response = await exchange.fetchOrderBook ('BTC/USDT')
    // ... some processing here ...
    return response
}

while (true) {
    const result = await tick ()
    console.log (result)
}

```

- 如果不理解速率限制器的内部工作方式，并且百分之百确定自己知道自己在做什么，请勿违反此规则。为了确保安全，请始终在函数和方法调用链中重复使用交易所实例，如下所示：

```python
// DO THIS INSTEAD:

async function tick (exchange) {
    const response = await exchange.fetchOrderBook ('BTC/USDT')
    // ... some processing here ...
    return response
}

const exchange = new ccxt.binance ({ enableRateLimit: true })
while (true) {
    const result = await tick (exchange)
    console.log (result)
}

```

### Cloudflare / Incapsula DDoS防护:
- 一些交易所部署了Cloudflare或者Incapsula来进行DDoS防护。在高负载时，你的IP可能会被临时屏蔽。有时他们甚至会限制整个国家或地区。这种情况下，它们的服务器通常会返回一个显示HTTP 40x错误的页面，或者运行一个浏览器的AJAX测试/验证码测试，并延迟页面的重新加载数秒钟。然后你的浏览器/指纹会被临时授权并被添加到白名单中，或者接收到一个HTTP cookie以供进一步使用。

- 以下是DDoS防护问题、速率限制问题或基于位置的过滤问题的最常见症状：

- 所有类型交易方法都出现“RequestTimeout”异常

- 捕获到带有HTTP错误代码400、403、404、429、500、501、503等的“ExchangeError”或“ExchangeNotAvailable”

- 存在DNS解析问题、SSL证书问题和低级连接问题

- 收到交易所的模板HTML页面而不是JSON

- 如果遇到DDoS保护错误，无法连接到特定交易所，则：- 使用代理（尽管这样会减缓响应速度）

- 请求交易所支持将您添加到白名单中

- 尝试使用不同地理区域的替代IP

- 在分布式服务器网络中运行 软件

- 将软件运行在靠近交易所的地方（同一国家，同一城市，同一数据中心，同一服务器架构，同一服务器）

## 加密货币市场：

- 这部分是确保我们能够正常执行交易的核心，CCXT 把交易所里的所有东西，分成 3 个概念：
- **货币（Currency）**
比如 BTC、USDT、ETH、SOL，告诉你能不能充币、能不能提币、提币手续费多少、最小提多少
- **市场（Market）= 交易对**
比如 BTC/USDT、ETH/USDT
告诉你这是现货 / 合约 / 期权、最小能下多少量、最大能下多少、价格最多保留几位小数、数量最多保留几位小数、手续费多少
**精度（Precision）+ 限制（Limits）= 下单必须遵守的规矩**
- 这是最最重要的部分，不遵守的话交易所直接拒单，导致下单失败

### 货币结构：

- 每个交易所是交易某种有价值的东西的地方。交易所可能使用不同的术语来称呼它们：”一种货币”，”一种资产”，”一枚代币”，”一只股票”，”一种商品”，”一种加密货币”，”法定货币”等等。通常称交易一种资产换取另一种资产的地方为”一个市场”，”一个符号”，”一个交易对”，_”一个合约”_等等。软件

- 在ccxt库中，每个交易所都提供多个市场。每个市场由两个或多个货币定义。市场的集合因交易所而异，为跨交易所和跨市场套利提供了可能性。

```python
{
    'id':       'btc',       // 引用交易所内部的字符串字面量
    'code':     'BTC',       // 统一的大写字符串字面量代码
    'name':     '比特币',   // 字符串，可读性强的名称，如果指定了的话
    'active':    true,       // 布尔值，货币状态（可交易和可提取）
    'fee':       0.123,      // 提款费用，平坦的
    'precision': 8,          // 小数位数的数量 "小数点之后"（取决于exchange.precisionMode）
    'deposit':   true        // 布尔值，是否可存款
    'withdraw':  true        // 布尔值，是否可提款
    'limits': {              // 在该市场上下单的价值限制
        'amount': {
            'min': 0.01,     // 订单数量应>最小值
            'max': 1000,     // 订单数量应<最大值
        },
        'withdraw': { ... }, // 提款限制
        'deposit': {...},
    },
    'networks': {...}        // 由统一网络标识符（ERC20，TRC20，BSC等）索引的网络结构
    'info': { ... },         // 交易所中未解析的原始货币信息
}

```

- 每个货币都是一个关联数组（也称为字典），包含以下键：

- id。货币在交易所内部的字符串或数字ID。货币ID在交易所内部用于在请求/响应过程中标识货币。

- code。特定货币的大写字符串代码表示。货币代码用于在ccxt库中引用货币（下面会解释）。

- name。货币的可读性强的名称（可以是大写和小写字符的混合）。

- fee。由交易所指定的提款费值。在大多数情况下，这意味着以相同货币支付的固定金额。如果交易所没有通过公共端点指定它，则fee可能是undefined/None/null或缺失。

- active。一个布尔值，表示当前是否可以交易或资金（存款或提取）该货币，详细信息请参见：active状态。

- info。关联数组，包含非常用市场属性，包括费用、汇率、限制和其他一般市场信息。每个特定市场的内部info数组都不同，其内容取决于交易所。

- precision。交易所在引用此货币时接受的结果的精度。此属性的值取决于exchange.precisionMode。

- limits。金额（成交量）、提款和存款的最小和最大值。

### 网络结构：

```python
{
    'id': 'tron', // 在交易所内部引用的字符串字面量
    'network': 'TRC20', // 统一的网络
    'name': '波场网络', // 字符串，可读性强的名称（如果有）
    'active': true, // 币种状态（可交易和可提现）的布尔值
    'fee': 0.123, // 提现费用，固定数额
    'precision': 8, // 小数点后的位数（取决于exchange.precisionMode）
    'deposit': true, // 是否允许存款的布尔值
    'withdraw': true, // 是否允许提款的布尔值
    'limits': { // 下单时的最小和最大值限制
        'amount': {
            'min': 0.01, // 订单数量必须大于min
            'max': 1000, // 订单数量必须小于max
        },
        'withdraw': { ... }, // 提款限制
        'deposit': { ... }, // 存款限制
    },
    'info': { ... }, // 来自交易所的原始未解析货币信息
}
```

- 每个网络都是一个关联数组（也称为字典），具有以下键：

- id。网络在交易所内的字符串或数字ID。在请求/响应过程中，交易所内部使用网络ID来标识网络。

- network。特定网络的大写字符串表示。在ccxt库中用于引用网络。

- name。网络的可读名称（可以是大写和小写字母的混合形式）。

- fee。交易所指定的提款费用。在大多数情况下，这意味着以同一货币支付的固定金额。如果交易所未通过公共端点指定它，则fee可能为undefined/None/null或缺失。

- active。一个布尔值，指示此货币当前是否可以交易或充提币。详细信息请参见active状态。

- info。非公共市场属性的关联数组，包括费用、汇率、限制和其他一般市场信息。内部信息数组因特定市场而异，其内容取决于交易所。

- precision。在引用该货币时交易所接受的精度。该属性的值取决于exchange.precisionMode。

- limits。金额（交易量）、提款和存款的最小和最大值。

###市场结构：

```python
{
    'id': 'btcusd', // 在交易所内部引用的字符串字面量
    'symbol': 'BTC/USD', // 交易对的大写字符串字面量
    'base': 'BTC', // 大写字符串，统一的基础货币代码，3个或更多字母
    'quote': 'USD', // 大写字符串，统一的报价货币代码，3个或更多字母
    'baseId': 'btc', // 基础货币的交易所特定ID，不统一。可以是任何字符串。
    'quoteId': 'usd', // 报价货币的交易所特定ID，不统一。
    'active': true, // 市场状态的布尔值
    'type': 'spot', // spot表示现货，future表示到期期货，swap表示永续互换，'option'表示期权
    'spot': true, // 市场是否是现货市场的布尔值
    'margin': true, // 市场是否是保证金市场的布尔值
    'future': false, // 市场是否是到期期货
    'swap': false, // 市场是否是永续互换
    'option': false, // 市场是否是期权合约
    'contract': false, // 市场是否是期货、永续互换或期权
    'settle': 'USDT', // 合约结算的统一货币代码，仅当`contract`为true时设置
    'settleId': 'usdt', // 合约结算的货币ID，仅当`contract`为true时设置
    'contractSize': 1, // 一个合约的大小，仅在`contract`为true时使用
    'linear': true, // 合约是否是线性合约（以报价货币结算）
    'inverse': false, // 合约是否是反向合约（以基础货币结算）
    'expiry': 1641370465121, // 以毫秒为单位的Unix到期时间戳，除了market['type']为`future`之外，都为未定义
    'expiryDatetime': '2022-03-26T00:00:00.000Z', // iso8601格式的合约到期时间
    'strike': 4000, // 可以行使看涨或看跌期权的价格
    'optionType': 'call', // 看涨或看跌的字符串，看涨期权表示具有购买权的期权，看跌期权表示具有出售权的期权
    'taker': 0.002, // 吃单者手续费率，0.002 = 0.2%
    'maker': 0.0016, // 挂单者手续费率，0.0016 = 0.16%
    'percentage': true, // 吃单者和挂单者手续费率是否为乘数或固定金额的布尔值
    'tierBased': false, // 手续费是否取决于交易等级（交易量）
    'feeSide': 'get', // 字符串字面量可以是'get'、'give'、'base'、'quote'、'other'
    'precision': { // 小数点后的位数
        'price': 8, // 四舍五入模式的整数或浮点数，如果交易所未提供，则可能缺失
        'amount': 8, // 整数，如果交易所未提供，则可能缺失
        'cost': 8, // 整数，很少有交易所实际提供
    },
    'limits': { // 在该市场上下单时的最小值和最大值限制
        'amount': {
            'min': 0.01, // 订单数量必须大于min
            'max': 1000, // 订单数量必须小于max
        },
        'price': { ... }, // 订单价格的相同min/max限制
        'cost': { ... }, // 订单成本（cost = price * amount）的相同限制
        'leverage': { ... }, // 杠杆的相同min/max限制
    },
    'info': { ... }, // 来自交易所的原**警告！有关费用的信息是实验性的，不稳定的，可能是部分的或根本不可用。**
    }

```

### 精度与限制：

- 不要将“限制”与“精度”混淆！ 精度与最小限制无关。8位数字的精度并不一定意味着最小限制为0.00000001。相反，最小限制为0.0001并不一定意味着精度为4。

- 示例：
```
(market['limits']['amount']['min'] == 0.05) && (market['precision']['amount'] == 4)
```

- 在此示例中，放置在市场上的任何订单的金额必须满足这两个条件：

- 金额的值应该 >= 0.05:

+ 好: 0.05, 0.051, 0.0501, 0.0502, ..., 0.0599, 0.06, 0.0601, ...
- 不好：0.04, 0.049, 0.0499

- 金额的精度应为4位小数：

+ 好: 0.05, 0.051, 0.052, ..., 0.0531, ..., 0.06, ...0.0719, ...
- 不好：0.05001, 0.05000, 0.06001

```
(market['limits']['price']['min'] == 0.019) && (market['precision']['price'] == 5)
```

- 在此示例中，放置在市场上的任何订单的价格必须满足这两个条件：

- 价格的值应该 >= 0.019:

+ 好: 0.019, ... 0.0191, ... 0.01911, 0.01912, ...
- 不好：0.016, ..., 0.01699

- 价格的精度应为5位小数或更低：

+ 好: 0.02, 0.021, 0.0212, 0.02123, 0.02124, 0.02125, ...
- 不好：0.017000, 0.017001, ...

```
(market['limits']['amount']['min'] == 50) && (market['precision']['amount'] == -1)
```

- 在这个例子中，两个条件都必须满足：

- amount值应该大于或等于50：

+ good: 50, 60, 70, 80, 90, 100, ... 2000, ...
- bad: 1, 2, 3, ..., 9

- 负的amount精度意味着amount应该是指定幂次的10的整数倍：

+ good: 50, ..., 110, ... 1230, ..., 1000000, ..., 1234560, ...
- bad: 9.5, ... 10.1, ..., 11, ... 200.71, ...

- precision和limits参数目前正在研发中，一些字段可能在一致化过程完成之前会有所缺失。这并不会影响大多数订单，但在非常大或非常小的订单的极端情况下可能会有影响。

### 关于精度和限制的说明
- 用户必须遵守所有限制和精度！订单的值应满足以下条件：

- 订单amount >= limits['amount']['min']

- 订单amount <= limits['amount']['max']

- 订单price >= limits['price']['min']

- 订单price <= limits['price']['max']

- 订单cost (amount * price) >= limits['cost']['min']

- 订单cost (amount * price) <= limits['cost']['max']

- amount的精度必须小于等于precision['amount']

- price的精度必须小于等于precision['price']

- 上述的值在一些交易所的API中可能会缺失或尚未实现。

### 格式化小数的方法
- 每个交易所都有自己的四舍五入、计数和填充模式。支持的舍入模式有：

- ROUND – 将最后几位小数舍入到指定精度

- TRUNCATE – 截取指定精度后的小数位

- 可以通过 exchange.precisionMode 属性设置小数精度计数模式。

### 精度模式
- 在 exchange['precisionMode'] 中支持的精度模式有：

- DECIMAL_PLACES – 计算所有数字的位数，99% 的交易所使用这种计数模式。在这种精密度模式中，market_or_currency['precision'] 中的数字指定小数点后要进行舍入或截断的位数。

- SIGNIFICANT_DIGITS – 仅计算非零数字的位数，一些交易所（如 bitfinex 和其他一些）实施这种计算小数位数的模式。在这种精度模式中，market_or_currency['precision'] 中的数字指定小数点后最后一个重要（非零）数字的第 N 位。

- TICK_SIZE – 一些交易所只允许使用特定值的倍数（例如 bitmex 和 ftx 使用此模式）。在这种模式下，market_or_currency['precision'] 中的数字指定用于舍入或截断的最小精度小数部分。

### 填充模式
- 支持的填充模式有：

- NO_PADDING – 大多数情况下的默认模式

- PAD_WITH_ZERO – 在精度之后添加零字符

- 大多数情况下，当用户下订单或发送提款请求时，用户不必关心精确的格式化，因为CCXT会在用户按照精度和限制的规则操作时为用户处理这些。然而，在某些情况下，精确格式化细节可能很重要，因此以下方法可能对用户很有用。

- 交易所基类包含decimalToPrecision方法，可帮助以不同的四舍五入、计数和填充模式格式化值。

```python
# 警告！`decimal_to_precision`方法容易受到`getcontext().prec`的影响！
def decimal_to_precision(n, rounding_mode=ROUND, precision=None, counting_mode=DECIMAL_PLACES, padding_mode=NO_PADDING):
```


- 请参阅以下文件，了解如何使用decimalToPrecision格式化字符串和浮点数的示例：

- Python：https://github.com/ccxt/ccxt/blob/master/python/ccxt/test/base/test_number.py

- Python 警告！decimal_to_precision方法容易受到getcontext().prec的影响！

- 为了方便用户，CCXT基类还实现了以下方法：

```python
def amount_to_precision (symbol, amount):
def price_to_precision (symbol, price):
def cost_to_precision (symbol, cost):
def currency_to_precision (code, amount):
```

- 每个交易所都有自己的精度设置，上述方法将根据交易所特定的精度规则格式化这些值，以一种可移植且与底层交易所无关的方式。为了实现这一点，在格式化任何值之前，必须先加载市场和货币。

- **在调用这些方法之前，确保 使用 exchange.loadMarkets() 加载市场！**

例如：

```python
exchange.load_markets()
symbol = 'BTC/USDT'
amount = 1.2345678  # BTC 的基本货币数量
price = 87654.321  # USDT 的报价货币价格
formatted_amount = exchange.amount_to_precision(symbol, amount)
formatted_price = exchange.price_to_precision(symbol, price)
print(formatted_amount, formatted_price)
```

- 更多描述 exchange.precisionMode 行为的实际示例：

```python
// 情况 A
exchange.precisionMode = ccxt.DECIMAL_PLACES
market = exchange.market (symbol)
market['precision']['amount'] === 8 // 小数点后最多 8 位数
exchange.amountToPrecision (symbol, 0.123456789) === 0.12345678
exchange.amountToPrecision (symbol, 0.0000000000123456789) === 0.0000000 === 0.0

// B 情况
exchange.precisionMode = ccxt.TICK_SIZE
market = exchange.market (symbol)
market['precision']['amount'] === 0.00000001 // 高达 0.00000001 精度
exchange.amountToPrecision (symbol, 0.123456789) === 0.12345678
exchange.amountToPrecision (symbol, 0.0000000000123456789) === 0.00000000 === 0.0

// C 情况
exchange.precisionMode = ccxt.SIGNIFICANT_DIGITS
market = exchange.market (symbol)
market['precision']['amount'] === 8 // 高达 8 个有效数字
exchange.amountToPrecision (symbol, 0.0000000000123456789) === 0.000000000012345678
exchange.amountToPrecision (symbol, 123.4567890123456789) === 123.45678
```

## 加载市场：

- 在大多数情况下，在访问其他 API 方法之前，您需要先加载特定交易所的市场列表和交易符号。如果您忘记加载市场，ccxt 库会在首次调用统一 API 时自动完成加载市场的工作。它会依次发送两个 HTTP 请求，第一个请求用于获取市场信息，第二个请求用于获取其他数据。因此，首次调用统一 CCXT API 方法（比如 fetchTicker、fetchBalance 等）会比后续的调用时间更长，因为它需要在交易所 API 中加载更多的市场信息。有关详细信息，请参见速率限制器注意事项。

- 为了手动预先加载市场，请在交易所实例上调用 loadMarkets() / load_markets() 方法。它返回一个按交易符号索引的市场关联数组。如果您想要更多地控制逻辑的执行，建议手动预加载市场。

```python
okcoin = ccxt.okcoinusd()
markets = okcoin.load_markets()
print(okcoin.id, markets)
```

- 除了市场信息外，loadMarkets() 调用还会加载交易所的货币信息，并将信息分别缓存在 .markets 和 .currencies 属性中。

- 用户也可以绕过缓存，直接调用统一方法从交易所端点获取这些信息，即 fetchMarkets() 和 fetchCurrencies()，尽管不建议终端用户使用这些方法。推荐的预加载市场的方式是调用 loadMarkets() 统一方法。然而，对于新的交易所集成，如果底层交易所具有相应的 API 端点，需要实现这些方法。

### 符号和市场标识
- 货币代码是一个包含三到五个字母的代码，例如 BTC, ETH, USD, GBP, CNY, JPY, DOGE, RUB, ZEC, XRP, XMR等等。一些交易所拥有更长的代码来表示奇特的货币。

- 符号通常是一个以斜线分隔的交易货币对的大写字符串。斜线前的货币通常被称为基准货币，斜线后的货币被称为报价货币。一些代表符号的例子是： BTC/USD, DOGE/LTC, ETH/EUR, DASH/XRP, BTC/CNY, ZEC/XMR, ETH/JPY。

- 市场标识在REST请求-响应过程中用于引用交易所内的交易对。市场标识集在每个交易所中是唯一的，不可以跨交易所使用。例如，BTC/USD货币对/市场在不同的流行交易所上可能有不同的标识，例如btcusd，BTCUSD，XBTUSD，btc/usd，42 (数字标识)，BTC/USD，Btc/Usd，tBTCUSD，XXBTZUSD。你不需要记住或使用市场标识，它们只是交易所内部实现中用于HTTP请求-响应目的的存在。

- ccxt库将不常见的市场标识抽象成符号，统一为一种常见格式。符号并不等同于市场标识。每个市场都通过对应的符号进行引用。符号在交易所之间是通用的，这使得它们适用于套利和其他许多应用。

- 有时候用户可能会注意到像 'XBTM18' 或 '.XRPUSDM20180101' 这样的 “奇特/罕见的符号”。符号不必须包含斜线或表示一对货币。符号中的字符串实际上取决于市场的类型（是现货市场还是期货市场，是暗池市场还是过期市场等等）。强烈不推荐尝试解析符号字符串，不应依赖符号格式，建议使用市场属性代替。

- 市场结构以符号和标识为索引。基础交易所类也有内置方法，可以通过符号访问市场。大多数API方法在第一个参数中需要传入一个符号。在查询当前价格、下订单等方面，通常要求指定一个符号。

- 在大多数情况下，用户将与市场符号一起使用。在这些词典中访问不存在的键时，将得到一个标准的用户异常。

### 加载市场和货币的方法：
```python
print(exchange.load_markets())

etheur1 = exchange.markets['ETH/EUR']          # 通过交易对获取交易所结构
etheur2 = exchange.market('ETH/EUR')           # 通过交易对以略微不同的方式获取相同的结果

etheurId = exchange.market_id('ETH/EUR')       # 通过交易对获取交易对的ID

symbols = exchange.symbols                     # 获取交易所支持的所有交易对列表
symbols2 = list(exchange.markets.keys())       # 与上一行代码相同

print(exchange.id, symbols)                    # 打印所有交易对

currencies = exchange.currencies               # 货币词典

kraken = ccxt.kraken ()
kraken.load_markets()

kraken.markets['BTC/USD']                      # 交易对 --> 交易所 （通过交易对获取交易所）
kraken.markets_by_id['XXRPZUSD'][0]            # ID --> 交易所 （通过ID获取交易所）

kraken.markets['BTC/USD']['id']  # 符号 → ID（通过符号获取ID）
kraken.markets_by_id['XXRPZUSD'][0]['symbol']  # ID → 符号（通过ID获取符号）

okcoinusd = ccxt.okcoinusd()
okcoinusd.load_markets()

okcoinusd.markets['BTC/USD']                    # symbol → 市场（根据符号获取市场）
okcoinusd.markets_by_id['btc_usd'][0]              # id → 市场（根据ID获取市场）

okcoinusd.markets['BTC/USD']['id']              # symbol → ID （根据符号获取ID）
okcoinusd.markets_by_id['btc_usd'][0]['symbol'] # id → 符号（根据ID获取符号）
```

### 致命一致性：

- 在各种交易所之间存在术语模糊性，这可能会导致新入行的交易者产生困惑。一些交易所将市场称为“交易对”，而其他交易所将符号称为“产品”。根据ccxt库的术语，每个交易所包含一个或多个交易市场。每个市场都具有ID和符号。大多数符号是基础货币和报价货币的组合。

- 交易所 → 市场 → 符号 → 货币

- 从历史上看，不同的交易对称用于指代相同的交易对。一些加密货币（如Dash）甚至在其运行的整个生命周期中多次更改其名称。为了在交易所之间保持一致性，ccxt库将针对符号和货币执行以下已知替换操作：

- XBT → BTC：XBT是较新的符号，但是BTC在各交易所中更常见，并且更像比特币（了解更多）。

- BCC → BCH：比特币现金的分叉通常有两个不同的符号名称：BCC和BCH。BCC是比特现金的一个误导性符号，它与比特币连接（BitConnect）混淆。ccxt库将BCC转换为BCH（一些交易所和聚合器将其搞混）。

- DRK → DASH：DASH曾被称为Darkcoin，然后变成了Dash（了解更多）。

- BCHABC → BCH：2018年11月15日，比特币现金第二次分叉，因此现在有BCH（用于BCH ABC）和BSV（用于BCH SV）。

- BCHSV → BSV：这是比特币现金SV分叉的常见替换映射（有些交易所称其为BSV，其他交易所称其为BCHSV，我们使用前者）。

- DSH → DASH：尽量不要混淆符号和货币。DSH（Dashcoin）与DASH（Dash）不同。一些交易所将DASH不一致地标记为DSH，ccxt库对此进行了更正（DSH → DASH），但只会在一些混淆了这两种货币的交易所上进行。大多数交易所都将它们标记正确。只要记住DASH/BTC与DSH/BTC不同即可。

- XRB → NANO：NANO是RaiBlocks的较新代码，因此，CCXT统一的API在需要的地方将旧的XRB替换为NANO。https://hackernoon.com/nano-rebrand-announcement-9101528a7b76

- USD → USDT：一些交易所，如Bitfinex，HitBTC和其他一些交易所，在其列表中将货币命名为USD，但实际上这些市场实际上是交易USDT。混淆可能来自符号名称的三个字母限制，或者可能是由其他原因引起的。在实际上交易的货币是USDT而不是USD的情况下，ccxt库将执行USD → USDT转换。然而，请注意，一些交易所同时具有USD和USDT符号，例如， Kraken拥有“USDT/USD”交易对。

### 关于致命一致性的注释：

- 每个交易所都有自己的国际货币符号的关联数组，并存储在exchange.commonCurrencies属性中。有时，用户可能会在代码中注意到带有大小写词和空格的奇异符号名称。在命名和货币编码冲突的解决规则中，这些名称背后的逻辑解释如下：

- 首先，我们收集有关所涉及货币代码的交易所自身提供的所有信息。它们通常在其API或其文档、知识库或其他地方的网站上有他们的币种清单的描述。

- 当我们确定与货币代码相关的每个特定加密货币时，我们在CoinMarketCap上查找它们。

- 所有竞争加密货币中市场资本化最大的那个获胜，并保留其货币代码。例如，HOT通常代表Holo或Hydro Protocol。在这种情况下，Holo保留代码HOT，而Hydro Protocol将其名称作为其代码，即Hydro Protocol。因此，可能会出现HOT/USD（对应于Holo）和Hydro Protocol/USD的交易对 - 这些是两个不同的市场。

- 如果特定加密货币的市值未知或不足以确定获胜者，我们还会考虑交易量和其他因素。

- 确定获胜者之后，将通过.commonCurrencies在冲突的交易所内正确重新映射和替换所有其他竞争货币的代码名称。

- 不幸的是，这是一个正在进行中的工作，因为每天都有新的货币上市，并且会不时添加新的交易所，因此，总的来说，这是一个永无止境的在快速变化的环境中进行自我修复的过程，实际上是以“实时模式”。我们对您发现的所有冲突和不匹配表示感谢。

### 对命名一致性的问题的疑问：
- 符号是否可能发生变化？

- 简而言之，是的，有时候会发生变化，但很少见。如果绝对需要并且无法避免，符号映射可以更改。然而，以前的符号更改都是为了解决冲突或分叉的问题。到目前为止，还没有出现过在CCXT中一个币的市值超过了另一个币的情况下，两个币具有相同的符号代码。

- 我们能否依赖于总是用相同的符号列出相同的加密货币？

- 多多少少地）首先，这个库还在不断发展中，它试图适应不断变化的实际情况，所以可能会出现我们将来会通过更改一些映射来修复的冲突。最终，许可证上说“不提供保证，请自行承担风险”。然而，我们不会随意更改所有的符号映射，因为我们明白其中的后果，并且我们也希望依赖于这个库，并且我们不喜欢破坏向后兼容性。

- 如果一个重要代币的符号被分叉或必须更改，那么控制权仍然掌握在用户手中。exchange.commonCurrencies属性可以在初始化或后期进行覆盖，就像任何其他交易所属性一样。如果涉及到重要的代币，我们通常会发布关于如何通过向构造函数参数添加几行代码保留旧行为的说明。

### 基础货币和报价货币的一致性：
- 这取决于你使用的交易所，但其中一些交易所的基础货币和报价货币的配对是颠倒的（不一致的）。它们实际上是把基础货币和报价货币搞错了（交换/互换了位置）。在这种情况下，你会看到解析的base和quote货币值与市场子结构中的未解析的info不同。

- 对于这些交易所，ccxt会进行更正，当解析交易所的回复时，会交换和规范化基础货币和报价货币的位置。这种逻辑在财务和术语上是正确的。如果你想避免混淆，记住以下规则：基础货币总是在斜线前，报价货币总是在斜线后，无论是任何符号和任何市场。

### 市场缓存强制重新加载：
- loadMarkets（）/ load_markets（）也是一种“脏”方法，带有在交易所实例上保存市场数组的副作用。您只需调用一次此方法，在交换机上的所有后续对该方法的调用都将返回本地保存（缓存）的市场数组。

- 当加载交易所市场后，您可以随时通过 markets 属性访问市场信息。此属性包含按符号索引的市场的关联数组。如果您需要在已加载市场之后强制重新加载市场列表，再次向相同的方法传递 reload = true 标志即可。

## 四种API范式：

- ccxt库为我们提供了四种类型的API，分别是隐式API、统一API、公共API和私有API；它们分别有其特定的功能和设计理念。

- 隐式API：你没有直接调用，但 CCXT 在后台自动帮你执行的 API 调用。

- 统一API：CCXT 为了抹平不同交易所的接口差异，提供的一套标准化、跨交易所通用的 API 接口。

- 公共API：不需要 API Key，任何人都能调用的接口，获取公开市场数据。

- 私有API：需要你的 API Key + 签名，只能访问你自己账户数据的接口。

## 隐式API：

### API方法/终端点：

- 每个交易所都提供了一组API方法。 API的每个方法被称为 终端点。 终端点是用于查询各种类型信息的HTTP URL。 所有终端点都返回JSON以响应客户端请求。

- 通常，交易所有一个用于获取市场列表的终端点，一个用于检索特定市场的订单簿的终端点，一个用于检索交易历史的终端点，以及用于下订单、取消订单、存款和提款的终端点等等… 基本上，你在一个特定交易所内执行的每种操作都有一个API提供的单独的终端点URL。

- 由于每个交易所的方法集不同，ccxt库实现了以下功能：

- 为所有可能的URL和方法提供的公共和私有API

- 一个支持一部分常见方法的统一API

- 终端点的URL在每个交易所的api属性中预定义。除非你正在实现新的交易所API（至少你应该知道你在做什么），否则你不必覆盖它。

- 大多数特定交易所的API方法是隐式的，这意味着它们在代码的任何地方都没有明确定义。该库实现了一种声明式的方法来定义隐式（非统一化）交易所API方法。

## 隐式API方法：

- API的每个方法通常有自己的终端点。库在.api属性中定义了每个特定交易所的所有终端点。在交易所实例的defineRestApi()/define_rest_api()中，针对.api终端点列表中的每个终端点，将会创建一个隐式的神奇方法（也称为部分函数或闭包）。这个操作对于所有交易所来说都是通用的。每个生成的方法都可以以camelCase和under_score表示法访问。

- 终端点定义是一个暴露给交易所的全部API URL列表。在交易所实例化时，该列表会被转换为可调用的方法。API终端点列表中的每个URL都会得到一个相应的可调用方法。这对于所有交易所都是自动完成的，因此ccxt库支持加密交易所提供的所有可能的URL。

- 每个隐式方法都有一个唯一的名称，该名称是根据.api定义构建的。例如，私有HTTPS PUT https://api.exchange.com/order/{id}/cancel终端点将具有相应的交易所方法名称.privatePutOrderIdCancel()/ .private_put_order_id_cancel()。公共HTTPS GET https://api.exchange.com/market/ticker/{pair}终端点将导致相应的方法名称.publicGetTickerPair()/ .public_get_ticker_pair()，依此类推。隐式方法接受一个参数字典，将请求发送到交易所，并返回 API 的特定于交易所的 JSON 结果原样，未解析。要传递参数，请将其明确地添加到字典中，键的名称等于参数的名称。对于上面的例子，这将类似于 .privatePutOrderIdCancel ({ id: '41987a2b-...' }) 和 .publicGetTickerPair ({ pair: 'BTC/USD' })。

- 与交易所一起工作的推荐方式不是使用特定于交易所的隐式方法，而是使用统一的 ccxt 方法。特定于交易所的方法应在没有对应的统一方法可用（尚未提供）的情况下使用。

- 要获取交易所实例的所有可用方法列表，包括隐式方法和统一方法，只需执行以下操作：

```python
print(dir(ccxt.kraken())) 
```

### 公共/私有 API:
- API URL 经常被分为两组方法集，称为用于市场数据的“公共 API”和用于交易和账户访问的“私有 API”。这些 API 方法组通常以“public”或“private”为前缀。

- 公共 API 用于访问市场数据，不需要任何身份验证。大多数交易所向所有人（在其限制速率下）公开提供市场数据。使用 ccxt 库，任何人都可以直接获取市场数据，无需在交易所注册，并且无需设置账户密钥和密码。

- 公共 API 包括以下内容：

- 交易对/货币对

- 价格源（汇率）

- 订单簿（L1、L2、L3…）

- 交易历史记录（已关闭订单、交易、执行）

- Ticker（实时价格 / 24 小时价格）

- 用于图表的 OHLCV 数据

- 其他公共终点网络

- 私有 API 主要用于交易和访问特定于账户的私有数据，因此需要进行身份验证。您必须从交易所获取私有 API 密钥。通常这意味着在交易所网站注册并为您的账户创建 API 密钥。大多数交易所需要个人信息或身份验证。完成 KYC 验证后，才允许进行交易。 私有 API 允许以下操作：- 管理个人帐户信息

- 查询账户余额

- 通过市价和限价单进行交易

- 创建存款地址和资金账户

- 请求提取法定货币和加密货币资金

- 查询个人的开放/关闭订单

- 查询保证金/杠杆交易的头寸

- 获取总账历史

- 在账户之间转移资金

- 使用商家服务

- 一些交易所以不同的名称提供相同的逻辑。例如，公共API通常也被称为市场数据、基础数据、市场、mapi、api、价格等等… 所有这些都表示一组方法，用于访问公共可用数据。私有API通常也被称为交易、交易所、tapi、账户等等…

- 一些交易所还提供一个商家API，允许您创建发票并接受来自客户的加密货币和法定货币支付。这种类型的API通常被称为商家、钱包、支付、ecapi（用于电子商务）。

- 要获取交换实例的所有可用方法列表，只需执行以下操作：软件

```python
print(dir(ccxt.kraken()))           # Python
```
- 仅合约和仅保证金

- 在这份文档中标记为仅合约或仅保证金的方法仅用于合约交易和保证金交易。尽管在其他类型的交易市场中也可能工作，但很可能返回无关的信息。

## 统一API：

- 统一的 ccxt API 是交易所之间常见方法的子集。它当前包含以下方法：

- fetchMarkets(): 从交易所获取所有可用市场的列表，并返回一个市场数组（包含 symbol、base、quote 等属性的对象）。某些交易所没有通过其在线 API 获取市场列表的方式。对于这些交易所，市场列表是硬编码的。

- fetchCurrencies(): 从交易所获取所有可用的货币，并返回一个关联的货币字典（包含 code、name 等属性的对象）。某些交易所没有通过其在线 API 获取货币的方式。对于这些交易所，货币将从市场对中提取或者硬编码。

- loadMarkets([reload]): 将市场列表作为按符号索引的对象返回，并将其缓存到交易所实例中。如果已经加载了缓存的市场，则返回缓存的市场，除非强制使用 reload = true 标志。

- fetchOrderBook(symbol[, limit = undefined[, params = {}]]): 获取特定市场交易对的 L2/L3 深度订单簿。

- fetchStatus([, params = {}]): 根据交易所实例中的固定信息或可用的 API 返回有关交易所状态的数据。

- fetchL2OrderBook(symbol[, limit = undefined[, params]]): 获取特定交易对的 2 级（按价格聚合）深度订单簿。

- fetchTrades(symbol[, since[, [limit, [params]]]]): 获取特定交易对的最近成交记录。

- fetchTicker(symbol): 获取特定交易对的最新行情数据。

- fetchBalance(): 获取余额。

- createOrder(symbol, type, side, amount[, price[, params]])

- createLimitBuyOrder(symbol, amount, price[, params])

- createLimitSellOrder(symbol, amount, price[, params])

- createMarketBuyOrder(symbol, amount[, params])

- createMarketSellOrder(symbol, amount[, params])

- cancelOrder(id[, symbol[, params]])

- fetchOrder(id[, symbol[, params]])

- fetchOrders([symbol[, since[, limit[, params]]]])

- fetchOpenOrders([symbol[, since, limit, params]]]])

- fetchCanceledOrders([symbol[, since[, limit[, params]]]])

- fetchClosedOrders([symbol[, since[, limit[, params]]]])

- fetchMyTrades([symbol[, since[, limit[, params]]]])

- fetchOpenInterest([symbol[, params]])
 
 ### 覆盖统一 API 参数:
 
 - 注意，统一 API 的大多数方法都接受一个可选的 params 参数。这是一个关联数组（字典），默认为空，其中包含要覆盖的参数。params 的内容是特定于交易所的，请查阅交易所的 API 文档以了解支持的字段和值。如果需要传递自定义设置或可选参数到统一查询中，请使用 params 字典。

 ```python
 # Python
params = {
    'foo': 'bar',       # exchange-specific overrides in unified queries
    'Hello': 'World!',  # see their docs for more details on parameter names
}
result = exchange.fetch_order_book(symbol, length, params)
 ```
 ### 分页：
 
 - 大多数的统一方法都会返回一个单一的对象或者一个普通数组（列表）的对象（交易、订单、交易记录等等）。然而，很少有交易所会一次性返回所有订单、所有交易、所有的K线数据、所有交易记录。大多数时候，它们的API都会限制输出的数量，只返回某个特定数量的最新对象。你不能通过一次调用获取从开始到现在的所有数据。实际上，很少有交易所会容忍或允许这样做。脚本语言

- 要获取历史订单或交易记录，用户需要分段或按页获取数据。分页通常意味着在循环中“逐个获取数据的部分”。

- 在大多数情况下，用户需要至少使用某种分页方式，以便在一致性上能得到预期的结果。如果用户没有应用任何分页，大多数方法将返回交易所的默认值，该默认值可能从历史开始，也可能是最近对象的子集。默认行为（无分页）是特定于交易所的！分页的方式通常在特定的方法中使用，如下所示：

- fetchTrades()

- fetchOHLCV()

- fetchOrders()

- fetchCanceledOrders()

- fetchClosedOrder()

- fetchClosedOrders()

- fetchOpenOrder()

- fetchOpenOrders()

- fetchMyTrades()

- fetchTransactions()

- fetchDeposit()

- fetchDeposits()

- fetchWithdrawals()

- 对于返回对象列表的方法，交易所可以提供一种或多种类型的分页方法。CCXT默认使用基于日期的分页，在整个库中使用的是以毫秒为单位的时间戳。

- 这是目前在CCXT统一API中使用的分页类型。用户提供一个以毫秒为单位的since时间戳，并提供一个limit参数来限制结果。为了逐页遍历感兴趣的对象，用户运行以下代码（下面是伪代码，可能需要根据具体的交易所覆盖一些交易所特定的参数）：

```python
if exchange.has['fetchOrders']:
    since = exchange.milliseconds() - 86400000  # 从当前时间倒推一天
    # 或者，从特定的起始日期获取
    # since = exchange.parse8601('2018-01-01T00:00:00Z')
    all_orders = []
    while since < exchange.milliseconds():
        symbol = None  # 更改为您的交易对
        limit = 20  # 更改为您的限制
        orders = await exchange.fetch_orders(symbol, since, limit)
        if len(orders):
            since = orders[len(orders) - 1]['timestamp'] + 1
            all_orders += orders
        else:
            break
```

- 还有诸多分页方式本文章就不一一例举了，感兴趣的读者可以查看官方文档；