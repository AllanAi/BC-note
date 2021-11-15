[Bancor 白皮书中文版](https://github.com/FIBOSIO/ibo.fo)

[公式推导](https://drive.google.com/file/d/0B3HPNP-GDn7aRkVaV3dkVl9NS2M/view?resourcekey=0-mbIgrdd0B9H8dPNRaeB_TA)

https://support.bancor.network/hc/zh-cn

> Bancor 协议采用「**恒定准备金率**（Constant Reserve Ratio）」策略促成价格发现，新代币的创建者需要抵押一定价值的「准备金」。
> $$
> CW = \frac{连接器代币余额}{智能代币总价值}
> $$
> 
>
> https://docs.qq.com/doc/DQ1VKUFRQWmJmbEx3

> 
> $$
> 代币新发行量 = 代币当前总供应量 \times ((1+\frac{支付的连接器代币数量}{连接器代币余额})^{CW}-1)
> $$
>
> $$
> 取出的连接器代币 = 连接器代币余额 \times (\sqrt[CW]{1+\frac{被销毁的智能代币数量}{代币当前总供应量}}-1)
> $$
>
> Bancor 白皮书中文版

> V2
>
> 通过使用**预言机(oracles)**缓和无常损失风险。
>
> Bancor V2打破了传统AMM，引入了预言机喂价AMM，从此AMM两边代币价值不用相等。V2采用预言机喂价，来调整代币两边的权重，也就是A代币数量*A价格不必等于B代币数量*B价格，把套利机会用预言机给磨平了。
>
> 该策略有两个明显缺陷：1/ 放弃了定价权；2/ 预言机喂价延迟可能会导致更多的套利。
>
> V2.1
>
> V2.1把BNT变成一种“现金流代币”，因为协议现在能够从与流动性提供者共同投资的BNT上产生收益，弹性BNT供应解决AMM难题。
>
> Bancor v2.1 为AMM提供了两大核心功能：
>
> 流动性保护（即无常损失保险）；单资产做市
>
> Bancor V2.1 交易的曲线和 Uniswap 一样，都是保持双边资产价值的 1:1 ——相同的策略也就意味着，流动性提供者将会有和 Uniswap 拥有一样的无常损失。Bancor V2.1 是如何应对的呢？
>
> Bancor V2.1 的做法是，**增发 BNT 补偿流动性提供者的无常损失**。
>
> 单边做市：
>
> Bancor V2.1 仅支持 BNT/ERC20 的流动性池，此举就是为了**实施弹性供应，以支持单边做市**。具体是怎么做的呢？
>
> 当流动性提供者提供 ETH 进行单边做市时，**Bancor V2.1 会按市场汇率增发等价值的 BNT 进入流动性池与用户的 ETH 配对。**
>
> **当流动性提供者提供 BNT 进行单边做市时，Bancor V2.1 会将用户的 BNT 存入流动性并销毁池中等量的、由协议拥有 BNT。**
>
> 当流动性提供者注入资产进行做市时，Bancor V2.1 会记录关键数据，以便在用户取会流动性时计算流动性提供者做市期间的无常损失。**在用户取回流动性时，协议将增发 BNT 用以补偿流动性提供者的无常损失。**
>
> https://docs.qq.com/doc/DQ2NrcVV4SXhjYVhq

https://blog.bancor.network/archive