# GMX share
## 前端及操作逻辑： Trade-Glp-Staking
<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-11-06-25.png?raw=true" /></center>

<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-11-04-40.png?raw=true" /></center>  
<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-11-07-07.png?raw=true" /></center>   

## 行业地位：

<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-11-08-41.png?raw=true" /></center>   
  
## 参考资料：  
https://www.notion.so/GMX-Technical-Overview-adcd9003194f444f81a28bcbecdde773     

## 项目历史  
1. Started XVIX in November 2020, 
2. Upgraded to Gambit on BSC in March 2021, 
3. Launched GMX in September 2021 on Arbitrum, 
4. GMX Update 1 (1 week after launch), Pricing improvements so traders can trade with smaller spreads, LINK, UNI and USDT added to GLP Pool to steer GLP further towards the 
S&P 500 for crypto vision, Integrations with aggregators, 
5. GMX Update 2 (3 months after launch), Launch on Avalanche as well as Arbitrum and BSC, 
Interface improvements, Launch of incentives program to encourage organic growth, Implementing fee discounts after community suggestions, 
Integration with Olympus Pro, Oversees GMX Blueberry Club delivery
## core dev：
https://github.com/xvi10  
<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-11-19-09.png?raw=true" /></center>
<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-11-15-28.png?raw=true" /></center>

## GMX设计思路-交易部分

GMX整体来讲是一个散户和LP对赌的期货交易所,CFD。 散户的交易不会影响到标记价格，不存在类似于现货交易所如Uni那样的价格发现曲线，交易的成交完全受标记价格控制。
GMX项目的去中心化程度相对较高，主要是用户的主要操作都在链上直接完成，如下图：

<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-16-12-43.png?raw=true" /></center>

GMX的用户主要有两类，分别是散户和GLP。散户的主要操作是开仓，平仓，swap等；GLP的主要操作是buyGLP和sellGLP来提供流动性和移除流动性。

同时GMX支持多种货币：WBTC,WETH,UNI,LINK,USDC,USDT,DAI,MIM,FRAX
但是支持多种token，并且还有非稳定币的token，在vault中应该如何正确记账？交易规则应该如何设置？

### GMX交易规则设置

- 开多，抵押品必须为WBTC，WETH，UNI，LINK这四种非稳定币，交易品种也必须与抵押品一致，平仓时，用户得到的利润也是WBTC，WETH，UNI，LINK
- 开空，抵押品必须是USDC,USDT,DAI,MIM,FRAX这五种稳定币，交易品种则只能是WETH-USD，WBTC-USD，平仓时，用户得到的利润也必须是USDC,USDT,DAI,MIM,FRAX
- 无论开多还是开空，盈亏的币种都与抵押品币种一致

思考：
1. 用稳定币开多为什么不行？同理，用非稳定币开空为什么不行？
2. 类似于MCDX，不同的币种如果分开交易，如何保证该币种的流动性深度足够，从而降低交易滑点？

最简单的方式可能是同时做多个市场，每一种市场一个币种。比如WBTC就是WBTC-USD交易对的币本位合约，USDC就是WBTC-USD交易对的U本位合约。这样做的好处是每一个币种都有清晰的市场，很好记账，但是坏处就是用户的流动性分散到多个市场里。用户交易时会发现深度很浅。

GMX的解决方案是提供一个混币池，给混币池做一个指数GLP，从而把用户提供的流动性全部集中起来，所有的币事实上都在这一个混币池里交易。

### GMX混币池Vault设计

作为一个混币池，需要给每个币定一个目标占比，这样可以根据占比来定一个指数：GLP

在Vault种，其设计为：

```jsx
// tokenWeights allows customisation of index composition
    mapping (address => uint256) public override tokenWeights;
WETH: 25% -- 28.65%
WBTC: 15% -- 14.26%
LINK: 5%  -- 2.49%
UNI:  1%  -- 0.81%
USDC: 30% -- 35.15% 
USDT: 9%  -- 6.06%
DAI:  12% -- 10.98%
MIM:  1%  -- 0%
FRAX: 2%  -- 1.58%
```

另一个关键问题是：混币池中，应该如何给每一种token记账？

由于是混币池，同时存在多种币种，稳定币，非稳定币等，设定一个内部记账用的token，usdg来表示相应token的价值，并规定1usdg=1刀。这样系统内部的所有账本都可以基于usdg来记账。

此时需要分析：vault中，token的来源：

- LP流动性提供者，通过任意一种token来购买指数GLP，从而给vault提供流动性
- 散户开多单，向Vault中提供WBTC，WETH，UNI，LINK这四种非稳定币，多单的盈亏也是这4种非稳定币
- 散户开空单，向Vault中提供USDC,USDT,DAI,FRAX,MIM这五种稳定币，空单的盈亏也是这5种稳定币
- 任何用户来swap，向vault种提供tokenA，按照预言机价格换成tokenB。可以认为是用户拿着tokenA卖给系统，换成系统内部记账的usdg，然后拿着系统内部记账的usdg买tokenB。此时流入tokenA，流出tokenB

<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-16-17-31.png?raw=true" /></center>

- 针对LP提供者，当他们buyGLP时，可以认为他们是拿着手上的WBTC，WETH等币，按照当时的WBTC，WETH等的价格先去vault里面购买usdg，然后拿着usdg这种内部记账的稳定币来按照当时的GLP价格购买GLP。
<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-16-17-59.png?raw=true" /></center>
如上图所示，用户拿着1 WETH来购买GLP，在Vault种，就是用WETH来换成vault内部记账的USDG。此时vault的WETH数量增加，WETH这种token对应的USDG债务增加。
则需要一个mapping来记录vault收到的token数量：`PoolAmounts(WETH⇒0.997 ether)`
还需要一个mapping来记录vault里该种token对应的USDG债务：`UsdgAmounts(WETH⇒3000 U)`
为了收集手续费，需要一个记录手续费的账本：`feeReserves(WETH⇒0.003 ether)`
    
- 针对散户开多单，散户拿着WETH，开10倍杠杆的多单，事实上就是散户拿1 ether，然后vaults中的LP借给散户9 ether，一共10 ether的头寸，在ETH-USD:3000 的价位开多单。  
也可以理解为：散户拿着1 ether，然后把这1 ether卖给vaults，换成等值的3,000 usdg，然后vaults中的LP借给散户 27,000 usdg，一共 30,000 usdg的头寸，此时该仓位的平均价格是3,000 usdg.此时用户的开仓手续费，含资金费用（0 USD）为 30 usdg。
该手续费30 U，折合ETH数量为0.01 ether
此时，需要记录vaults中LP借给散户的27,000+30 USDG债务，即`guranteedUsd(WETH ⇒ 27,030 U)`
需要记录vaults中用户实际转账过来的WETH数量，即`poolAmounts(WETH ⇒ 0.99 ether)`
同时需要考虑到vaults中有足够的WETH来兑现散户的头寸，即`reserveAmounts(WETH ⇒ 10 ether)`, 要求是用于兑现所有散户的头寸的reserve WETH数量必须小于vaults中持有的WETH数量 poolAmounts
    
- 针对散户开空单，散户拿着3,000 USDC,开10倍的空单，也就是vaults中的LP借给散户27,000 USDC, 一共30,000 USDC的头寸，在ETH-USD:3000 的价格开空单。
也可以理解为：散户拿着3,000 USDC，然后把这3,000 USDC卖给vaults，换成等值的3,000 usdg，然后vaults中的LP借给散户 27,000 usdg，一共 30,000 usdg的头寸，此时该仓位的平均价格是3,000 usdg.此时用户的开仓手续费，含资金费用（0 Usd）为 30 usdg。
需要考虑到vaults中有足够的USDC来兑现散户的头寸，即`reserveAmounts(USDC⇒ 30,000 ether)`, 要求是用于兑现所有散户的头寸的reserve USDC数量必须小于vaults中持有的USDC数量 poolAmounts
由于开空单，只能是稳定币, 这里为了解决用USDC开ETH-USD空单和用USDT开ETH-USD空单，FRAX开ETH-USD空单的问题，如果还是按照开多单的思路，就会导致流动性分散，这里为了集中流动性，把统一成了一个全局的稳定币头寸，按照不同的标的来记录，`globalShortSize(ETH⇒30,000)`
### Glp定价
GLP最重要的是对于GLP的定价

$$
GLP_{price}=\frac{\sum AUM_{token}}{GLP_{totalSupply}} 
$$

$$
AUM_{stableToken}=PoolAmount_{token}\times Price_{token}
$$

$$
\begin{align*}
AUM_{NonStableToken} & = PoolAmount\times Price+PnL_{long}+PnL_{short}\\
PnL_{long} & = GuranteedUSD-ReserveAmount\times Price\\
PnL_{short} &= \pm Size_{globalShort}\times \frac{\left | Price-avgPrice_{globalShort} \right |  }{avgPrice_{globalShort}}\\
\end{align*}
$$

对于short部分：

$$
\begin{align*}
Price>avgPrice_{globalShort},User 亏损,LP 盈利,PnL_{short}>0\\
Price &lt avgPrice_{globalShort},User 盈利,LP 亏损,PnL_{short} &lt 0\\
\end{align*}
$$

对于long部分：

$$
{\begin{align*}
GuranteedUSD &: \text{用户开仓时，杠杆数超过1，而向LP借款的部分}\\
ReserveAmount\times Price&: \text{LP预留给用户用以实现仓位的部分的当前时刻价值}\\
ReserveAmount &: \text{用户开平仓时，那个时刻的仓位价值与那个时刻的价格比，即为LP预留的Token数量}\\  
\end{align*}}  
$$

在合约中，对于GLP的定价部分在GlpManager中的getAum函数中：
```js
function getAum(bool maximise) public view returns (uint256)
第一步：拿到vault中支持的所有token列表
第二步：设定Aum的初值为aumAddition，该aumAddition值由系统管理员设置
第三步：遍历所有的token列表，拿到某一个token，判断该token是否是白名单中的token
第四步：如果该token不是白名单的token，则跳过该token，不纳入统计范围
第五步：根据传入的参数，拿到该token的最大价格或者是最小价格，作为该token的统计价格
第六步：拿到该token的poolAmount作为该token的存在数量
第七步：拿到该token的小数位数
第八步：如果该token是稳定币，则aum+=poolAmount*Price
第九步：如果该token不是稳定币，则统计该token的价值时，
还需要考虑系统的在该token上的未实现盈亏，分别考虑多头和空头部分
第十步：需要首先拿到该token的globalShortSize，即vault在该token上的全局空仓头寸。
第十一步：如果该空仓头寸大于0，则拿到该空仓头寸的平均价格avgPrice，计算delta=size*|price-avgPrice|/avgPrice
如果当前价格比avgPrice价格高，则用户亏钱，LP赚钱，则aum+=delta
如果当前价格比avgPrice价格低，则用户赚钱，LP亏钱，则aum-=delta
第十二步：考虑多头部分，aum+=guranteedUsd[token]，即将用户从LP这里借的钱也算作LP的资产
第十三步：拿到系统预留下的token数量，reserveAmount
第十四步：从poolAmount中扣除reserveAmount后，在乘以price得到该token的净价值
第十五步：最后从aum中扣除aumDeduction部分，作为所有token的总价值
```
除了GLP的定价外，另一个值得关注的点是：具体购买/销毁GLP的流程

反映在合约中，即为addLiquidity和removeLiquidity

Add Liquidity:
<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-16-00-15.png?raw=true" /></center>

removeLiquidity
<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-16-00-28.png?raw=true" /></center>

## 风控机制
### 最大开仓限制
在increasePosition方法中，针对开仓的限制主要有：

1. `nextLeverage >= minLeverage`， 即开仓后新的杠杆要大于最小杠杆数
2. `position.collateral >= fee`，即新仓位的抵押品价值要大于手续费，含资金费
3. 新仓位不能立马被清算
4. `reservedAmounts[_token] <= poolAmounts[_token]`，这里的限制事实上就是对于新开仓位的最大开仓限制：

原因如下：
reserveAmounts记录的是开这个仓位的用户，该仓位的总价值兑换成`collateral token`的数量。也就是说GMX里并没有针对这单一的一个仓位进行限制，而是针对全局的所有仓位，所有该种collateral仓位的总和对应的`collateral token`数量即`reserveAmounts`不能超过Vaults里面存有的该种token的总量，`poolAmounts`

### 开仓保证金计算逻辑，限仓，限杠杆
限杠杆：1~max50
限仓：reservedAmounts[_token] <= poolAmounts[_token]
保证金计算逻辑：
对于做多，position仓位的更新过程如下：
如果是该用户针对该种token的首次做多，则首先更新仓位的平均价格，为indexToken的maxPrice。如果不是首次做多，则需要先计算一个仓位的平均价格，即现在的开仓价格*仓位的总头寸/(总头寸+delta) 

$$
\begin{align*}
Price_{avg} & = \frac{ Price \times Size}{Size+\Delta}\\
\Delta & = Size^{before} \times \frac{\left | Price - Price_{avg}^{before} \right | }{Price_{avg}^{before}}\\
Size&= Size^{before}+\delta Size
\end{align*}
$$

$$
Price_{avg}=\frac{Size}{\frac{Size^{before}}{Price^{before}}+\delta \times \frac{1}{Price}}
$$

然后更新仓位的collateral，即以U计价的抵押品价值

$$
\begin{align*}
\delta token & = tokenBalance^{after}-tokenBalance^{before}\\
collateral&=collateral+\delta token \times Price_{min}- fee_{margin}\\
fee_{margin}&=fee_{position}+fee_{funding}\\
fee_{position}&=\delta Size\times 0.1\%\\
fee_{funding}&=Size^{before}\times\frac{FundingRate_{acc}-FundingRate_{entry}}{1000000} 
\end{align*}
$$

然后分别更新仓位的entry和size，time等：

$$
\begin{align*}Size & = Size^{before}+\delta Size\\entryFundingRate&=FundingRate_{acc}\\lastIncreasedTime&=now\end{align*}
$$

最后是更新仓位的reserveAmount：

$$
\begin{align*}\delta reserve & = \frac{\delta Size}{Price_{min}}\\reserve & = reserve^{before}+\delta reserve \end{align*}
$$

更新完Position的仓位数据后，对于做多，还需要更新如下账本：

针对guranteedUsd账本，用于记录该种token对应的头寸与以U计价的保证金的差值

$$
\begin{align*}
guranteedUSD_{token}^{after} & = guranteedUSD_{token}^{before}+\Delta\\
\Delta & = \delta Size+fee_{margin}-\delta token \times  Price_{min}
\end{align*}
$$

针对PoolAmount账本，手续费被认为是不属于poolAmount中，因为margin fee在仓位的保证金价值里扣除的，而poolAmount只追踪仓位的保证金token数量。所以这里需要减去手续费对应的token数量。

$$
\begin{align*}poolAmount_{token} & = poolAmount_{token}^{before}+\Delta\\\Delta &= \delta token-\frac{fee_{margin}}{price_{max}}
\end{align*}
$$

### 平仓盈亏计算：指数价格，标记价格，手续费
手续费计算：
累计资金费率：

$$
\begin{align*}FoundRate_{acc} & = FoundRate_{acc}^{before}+\Delta \\\Delta & = 100 \times \frac{now-time_{last}}{8hour}\times \frac{reserveAmount_{token}}{poolAmount_{token}}  \end{align*}
$$

然后是计算margin的手续费：

$$
\begin{align*}fee_{margin}&=fee_{position}+fee_{funding}\\fee_{position}&=\delta Size\times 0.1\%\\fee_{funding}&=Size^{before}\times\frac{FundingRate_{acc}-FundingRate_{entry}}{1000000} \end{align*}
$$

平仓盈亏计算：

对于用户，用户可以部分平仓，也可以全平。部分平仓时，用户可以指定需要取出的coll数量,用USD计价，即$\delta coll$和需要平掉的仓位大小$\delta size$。

对于部分平多：

利用仓位对应的indexToken，来计算未实现盈亏delta

$$
\begin{align*}\Delta = Size^{before}\times \frac{\left | price-price_{avg}^{before} \right | }{price_{avg}^{before}} \end{align*}
$$

更新realisedPnL值和保证金coll值：

$$
\begin{align*}
USD^{out} &=\delta coll+\Delta \times \frac{\delta Size}{Size^{before}}  & \text{ 用户获利： } price\ge  price_{avg}^{before} \\
coll&=coll^{before} - \delta coll & \text{ 用户获利： } price\ge  price_{avg}^{before} \\
realisedPnL&=realisedPnL^{before}+ \Delta \times \frac{\delta Size}{Size^{before}}  & \text{ 用户获利： } price\ge price_{avg}^{before} \\
\end{align*}
$$

$$
\begin{align*}
USD^{out}&=\delta coll  & \text{ 用户亏损： } price<price_{avg}^{before} \\
coll&=coll^{before}- \Delta \times \frac{\delta Size}{Size^{before}} - \delta coll& \text{ 用户亏损： } price<price_{avg}^{before} \\
realisedPnL&=realisedPnL^{before}- \Delta \times \frac{\delta Size}{Size^{before}}  & \text{ 用户亏损： } price<price_{avg}^{before} \\
\end{align*}
$$

其中，USDout就是平仓时，需要由金库转给用户的USD值。

然后是扣除手续费，如果USDout大于手续费，则直接从USDout里面扣除，如果USDout小于手续费，则从保证金里扣除

$$
\begin{cases}USD^{out}&=USD^{out}-fee  & \text{ if:   } USD^{out}>fee \\coll&=coll-fee & \text{ if:   } USD^{out}\le fee \\\end{cases}
$$

然后是更新头寸和entryFoundingRate，

$$
\begin{align*}Size & = Size^{before}-\delta Size\\FoundingRate_{entry} & = FoundingRate_{acc}\\
\end{align*}
$$

对于做多，此时还需要更新guranteedUSD表

$$
\begin{align*}guranteedUSD_{token}^{after} & = guranteedUSD_{token}^{before}+\Delta\\\Delta & = coll^{before}-coll^{after}-\delta Size\end{align*}
$$

在给用户转账USDout前，需要折算USDout等价的token，并将其更新到PoolAmount账本中

$$
\begin{align*}poolAmount_{token} & = poolAmount_{token}^{before}-\Delta\\\Delta &= \frac{USD^{out}}{price_{max}} \end{align*}
$$

最后把折算好的token数量打给用户即可。

## 预言机处理，指数价格，标记价格合成
GMX有一套复杂的逻辑来根据index price计算mark price，下述的分析是基于目前GMX运行的状态
### index price
目前的index price的数据来源有两块：
第一部分是chainlink的数据源；
第二部分是项目方自己向合约里面输入的price。即fastPriceFeed。
```js
Function: setCompactedPrices(uint256[] _priceBitArray) ***

MethodID: 0x1d4e3740
[0]:  0000000000000000000000000000000000000000000000000000000000000020
[1]:  0000000000000000000000000000000000000000000000000000000000000001
[2]:  00000000000000000000000000000000000027f900003c3f002d9cf4029d04eb
```
### mark price
通过index price合成mark price，GMX自定义了一套算法，并且算法的参数可以实时修改。

1.针对稳定币：USDC,USDT,MIM,FRAX,DAI 
它的计算逻辑是从chainlink里拿到该稳定币的价格，如果该价格在[0.95-1.05]之间，就认为该稳定币的价格就是1. 
如果chainlink里拿到的稳定币价格不在[0.95-1.05]之间，则直接返回从chainlink里拿到的价格
2.非稳定币：WBTC，WETH
它的计算逻辑是：
- 从chainlink里拿到WETH的价格refPrice
- 把refPrice传到fastPriceFeed合约里，进行如下计算：
    - 计算minPrice = refPrice * (1 - 2.5%)
    - 计算maxPrice = refPrice * (1 + 2.5%)
    - 从fastPriceFeed里拿到最新updated的价格：fastPrice
    - 如果fastPrice在[minPrice, maxPrice]之间，则返回fastPrice
    - 如果fastPrice < minPrice,  maximise ? refPrice : minPrice
    - 如果fastPrice > maxPrice, maximise ? maxPrice : refPrice

3.非稳定币：UNI，LINK
对于UNI和LINK，它的基本计算逻辑与WETH，WBTC的逻辑一致，只是在返回的price上要额外乘以一个系数：正负 0.07%
$$
maximise ? price * 1.0007 : price * 0.9993
$$
项目方设定了一个机器人，让机器人每两个块，调用一次setCompactedPrices，来更新对应的token的价格，只包括WBTC,WETH,UNI,LINK的价格，不包括稳定币的价格. 如果两次更新价格的时间间隔超过了300秒，则不使用fastPriceFeed，让其直接返回chainlink的价格。

## staking 套娃机制
在GMX里面设计了很多种token，GMX，GLP，exGMX等。其质押的核心逻辑还是masterchef方式。在多种token的交互中，主要的设计思路是质押，托管，兑现（vesting）
<center><img src="https://github.com/Dapp-Learning-DAO/Dapp-Learning-Arsenal/blob/main/images/defi/GMX/2022-09-03-11-44-54.png?raw=true" /></center>
质押逻辑：
GMX related:
    1. 首先质押GMX，得到sGMX，同时得到奖励代币esGMX
    2. 拿sGMX质押到Bonus池子里，得到sbGMX，同时得到奖励代币bnGMX
    3. 拿得到的sbGMX和奖励的bnGMX分别质押到手续费池子里，得到sbfGMX，获得奖励代币WETH，即手续费分成
    4. 拿奖励代币esGMX配上sbfGMX代币，到GMX 兑换池中兑换成GMX

GLP related:
    1. 用ETH购买GLP，得到GLP
    2. 用GLP质押到手续费池子中，得到fGLP,同时获得手续费奖励WETH
    3. 用得到的fGLP质押到GLP池子中，得到fsGLP，同时获得奖励代币esGMX
    4. 拿奖励代币esGMX配上fsGLP，到GMX 兑换池中兑换成GMX
