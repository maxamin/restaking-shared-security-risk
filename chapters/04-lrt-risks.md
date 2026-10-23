# Liquid Restaking Token Risks

Liquid restaking tokens (LRTs) represent restaked positions and are traded and used in DeFi. They stack risks.

## Risk stack

```
DeFi lending using LRT collateral
  ↑ LRT price / liquidity (depeg risk)
  ↑ LRT protocol contracts and strategy (operator choice, AVS choice)
  ↑ restaking protocol contracts and slashing
  ↑ LST or native staking
  ↑ Ethereum consensus
```

Each layer has its own contracts, upgrade keys, oracles and delays. A failure at any layer propagates upwards: a slashing event lowers the LRT's backing, which lowers its market price, which triggers liquidations in lending markets that accepted it.

## Specific concerns

- **Withdrawal delays:** restaking withdrawals take time (staking exit queue + restaking delays). In stress, LRTs trade at discounts.
- **Oracle pricing:** pricing LRTs by exchange rate ignores market depeg; pricing by thin DEX markets invites manipulation (see **oracle-security-engineering**).
- **Looping:** users borrow against LRTs to buy more LRTs, amplifying liquidations during depeg.
- **Points-driven deposits:** incentive programmes attract capital that leaves quickly.
- **Smart contract risk** at each layer; upgrade keys at each layer.

## Why LRTs depeg

An LRT's redemption value is its backing divided by supply. Its market price can drift below that for reasons unrelated to slashing: when many holders want out at once, the only immediate exit is a DEX pool, and the redemption path takes days or weeks. Arbitrageurs who buy at a discount and redeem must lock capital for the whole delay, so they demand a discount proportional to that delay and to their cost of capital. Longer queues mean deeper discounts. Liquid staking tokens have shown this pattern during past market stress; LRTs add more layers of delay.

| Pricing method | Behaviour in a depeg | Manipulation risk |
|---|---|---|
| Exchange rate (backing / supply) | ignores market price; liquidations may come too late | low, unless the rate source is manipulable |
| DEX spot price | reacts immediately | high in thin pools |
| TWAP of DEX price | lags | medium |
| min(exchange rate, robust market price) | conservative | depends on market source |

## Defensive measures for integrators

- Conservative collateral factors and supply caps for LRTs.
- Price feeds that use min(market, exchange rate).
- Monitoring of LRT liquidity depth and withdrawal queue length.
- Exit liquidity analysis: how much LRT can be sold for how much slippage, and is that enough for your liquidations?

## Worked example

> **Execution status:** Steps 1–2 were executed with Python 3 while writing this chapter; the liquidation counts quoted below were observed with `random.seed(1)`. Steps 3–4 were not executed.

Simulate a depeg on a lending market with looped positions and count the liquidation cascade. All numbers are illustrative; replace them with data for a real market you analyse.

**Step 1: positions.** Each looper deposits LRT, borrows ETH at a loan-to-value ratio, buys more LRT, and repeats. With a target LTV of 0.8 and liquidation threshold 0.85, leverage approaches 1 / (1 − 0.8) = 5×.

```python
import random
random.seed(1)
LT = 0.85
# (collateral in LRT, debt in ETH); LTVs between 0.70 and 0.82
positions = [(c, c * random.uniform(0.70, 0.82)) for c in [random.uniform(10, 200) for _ in range(500)]]

def liquidations(price):
    liq = [(c, d) for c, d in positions if d / (c * price) > LT]
    return len(liq), sum(c for c, _ in liq)

for p in [1.00, 0.98, 0.95, 0.92, 0.90]:
    n, coll = liquidations(p)
    print(f"price {p:.2f}: {n:3d} positions liquidatable, {coll:8.0f} LRT to sell")
```

**Step 2: read the output.** With `random.seed(1)` you should see nothing at 1.00 and 0.98, 56 of 500 positions at 0.95, 179 at 0.92 and 246 at 0.90. Small depegs flip leveraged positions quickly because they sit near the threshold.

**Step 3: feed back into price.** Liquidators sell seized LRT into a DEX pool. Model the pool as constant product with, say, 5,000 LRT and 5,000 ETH; selling `x` LRT moves price to roughly `5000² / (5000 + x)²` relative to the start. Loop: compute liquidations at the current price, sell the collateral, update the price, repeat until nothing new is liquidatable. Note the final price and total liquidated.

**Step 4: test defences.** Re-run with a supply cap that limits total LRT collateral to 30% of pool depth, and with an LTV cap of 0.7. Compare final prices. This is the quantitative argument for conservative parameters.

## Common pitfalls

- **Exchange-rate oracles with no market check.** Positions stay "healthy" while their collateral cannot be sold at that price.
- **Thin-pool spot prices.** Cheap to manipulate, as many oracle incidents show.
- **Ignoring exit queue length.** It sets the depeg arbitrageurs demand.
- **Collateral caps above liquidation capacity.** If liquidators cannot sell, bad debt follows.
- **Not tracking upgrade keys at each layer.** One upgrade anywhere in the stack can change backing.

## Exercises

1. For one LRT, list every contract and upgrade key in its risk stack.
2. Simulate a 10% depeg on a lending market with looping positions and compute liquidations.
3. Design an oracle for an LRT that handles depeg safely.
4. Complete Step 3 of the worked example and plot price against liquidation round.

## Further reading

- L2BEAT-style risk frameworks for restaking (search current community dashboards)
- Ethereum staking withdrawal queue: <https://ethereum.org/en/staking/withdrawals/>
- Uniswap v2 whitepaper (constant-product pricing): <https://docs.uniswap.org/contracts/v2/concepts/protocol-overview/how-uniswap-works>
- See **oracle-security-engineering** in this wiki.
