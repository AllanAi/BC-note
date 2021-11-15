术语 https://docs.aave.com/developers/glossary

不同资产风险等级 https://docs.aave.com/risk/asset-risk/risks-per-asset

根据风险等级设置相关参数 https://docs.aave.com/risk/asset-risk/risk-parameters

| Name                  | Symbol | Collateral | Loan To Value | Liquidation Threshold | Liquidation Bonus | Reserve Factor |
| --------------------- | ------ | ---------- | ------------- | --------------------- | ----------------- | -------------- |
| **Stablecoins**       |        |            |               |                       |                   |                |
| Binance USD           | BUSD   | no         | -             | -                     | -                 | 10%            |
| DAI                   | DAI    | yes        | 75%           | 80%                   | 5%                | 10%            |
| Gemini Dollars        | GUSD   | no         | -             | -                     | -                 | 10%            |
| RAI                   | RAI    | no         | -             | -                     | -                 | 20%            |
| Synthetix USD         | SUSD   | no         | -             | -                     | -                 | 20%            |
| True USD              | TUSD   | yes        | 75%           | 80%                   | 5%                | 10%            |
| USDC                  | USDC   | yes        | 80%           | 85%                   | 5%                | 10%            |
| Tether                | USDT   | no         | -             | -                     | -                 | 10%            |
| **Other Assets**      |        |            |               |                       |                   |                |
| Aave                  | AAVE   | yes        | 50%           | 65%                   | 10%               | 0%             |
| Basic Attention Token | BAT    | yes        | 70%           | 75%                   | 10%               | 20%            |
| Balancer              | BAL    | yes        | 55%           | 65%                   | 10%               | 20%            |
| Curve DAO             | CRV    | yes        | 40%           | 55%                   | 15%               | 20%            |
| Enjin                 | ENJ    | yes        | 55%           | 60%                   | 10%               | 20%            |
| Ethereum              | ETH    | yes        | 80%           | 82.5%                 | 5%                | 10%            |
| Kyber Network         | KNC    | yes        | 60%           | 65%                   | 10%               | 20%            |
| Chainlink             | LINK   | yes        | 70%           | 75%                   | 10%               | 20%            |
| Decentraland          | MANA   | yes        | 60%           | 65%                   | 10%               | 35%            |
| Maker                 | MKR    | yes        | 60%           | 65%                   | 10%               | 20%            |
| Republic Protocol     | REN    | yes        | 55%           | 60%                   | 10%               | 20%            |
| Ren Filecoin          | RenFIL | no         | -             | -                     | -                 | 35%            |
| Synthetix             | SNX    | yes        | 15%           | 40%                   | 10%               | 35%            |
| Uniswap               | UNI    | yes        | 60%           | 65%                   | 10%               | 20%            |
| Wrapped BTC           | WBTC   | yes        | 70%           | 75%                   | 10%               | 20%            |
| Wrapped ETH           | WETH   | yes        | 80%           | 82.5%                 | 5%                | 10%            |
| Sushi Bar             | XSUSHI | yes        | 25%           | 45%                   | 15%               | 35%            |
| Yearn YFI             | YFI    | yes        | 40%           | 55%                   | 15%               | 20%            |
| 0x                    | ZRX    | yes        | 60%           | 65%                   | 10%               | 20%            |

BUSD、GUSD、RAI、SUSD、USDT等不能作为抵押物进行借贷。

AAVE虽然支持的资产比Compound多，但是设置了更加严格的风险参数。

> **Loan to Value**
>
> For each wallet the maximum LTV is calculate as the weighted average of the LTVs of the collateral assets and their value:
> $$
> MaxLTV=\frac{\sum Collateral_i\ in\ ETH \times LTV_i}{Total\ Collateral \ in\ ETH}
> $$
> **Liquidation Threshold**
>
> The delta between the Loan-To-Value and the Liquidation Threshold is a safety cushion for borrowers.
>
> For each wallet the Liquidation Threshold is calculate as the weighted average of the Liquidation Thresholds of the collateral assets and their value:
> $$
> Liquidation\ Threshold = \frac{\sum Collateral_i\ in\ ETH \times Liquidation\ Threshold_i}{Total\ Collateral \ in\ ETH}
> $$
> **Health Factor**
> $$
> H_f = \frac{\sum Collateral_i\ in\ ETH \times Liquidation\ Threshold_i}{Total\ Borrows \ in\ ETH}
> $$
> **Reserve Factor**
>
> The reserve factor allocates a share of the protocol's interests to a collector contract as reserve for the ecosystem.

历史使用率 https://docs.aave.com/risk/liquidity-risk/historical-utilization

稳定币的利用率高，其他资产则比较低。即大多数用户愿意抵押非稳定币来借贷稳定币。

利率模型 https://docs.aave.com/risk/liquidity-risk/borrow-interest-rate

和Compound类似，都是拐点型。
$$
if\ U < U_{optimal}: R_t = R_0 + \frac{U_t}{U_{optimal}}R_{slope1} \\
if\ U \geq U_{optimal}: R_t = R_0 + R_{slope1} + \frac{U_t-U_{optimal}}{1-U_{optimal}}R_{slope2}
$$
同时支持稳定利率，稳定利率中的$R_0$是通过预言机获取的各平台的借款利率的加权平均值。



合约概览 https://docs.aave.com/developers/the-core-protocol/protocol-overview



支持 FlashLoan
