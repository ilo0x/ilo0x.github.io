---
title: "Concentrated Liquidity: A Practical Framework"
date: 2026-02-22 12:00:00 +0100
categories: [DeFi, Research]
tags: [clamm, liquidity, amm, uniswap, aerodrome]
description: "An overview of concentrated liquidity mechanics, rebalancing strategies, and the tools I've built to analyze them."
math: true
---

## The Problem with Idle Capital

Most liquidity in DeFi sits in wide ranges, earning close to nothing. Concentrated liquidity AMMs like Uniswap V3 and Aerodrome changed this by letting LPs focus their capital into narrow price bands — but this introduced a new set of problems.

> "The tighter your range, the higher your fee capture — but the faster you get rekt when price moves against you."

The core tension is simple: **capital efficiency vs. impermanent loss**. Every LP position is a bet on where price will stay.

---

## How CLAMMs Work

In a traditional `x * y = k` AMM, liquidity is spread uniformly across all prices from 0 to ∞. In a concentrated liquidity AMM, you choose a range $[p_a, p_b]$ and your liquidity only earns fees while the price $p$ satisfies:

$$
p_a \leq p \leq p_b
$$

The narrower the range, the more your position behaves like **leveraged liquidity**. The effective multiplier can be approximated as:

$$
m = \frac{\sqrt{p_b} - \sqrt{p_a}}{\sqrt{p_b} - \sqrt{p}}
$$

### A Quick Example

| Range Width | Fee Multiplier |  IL Risk  |
| :---------- | :------------: | :-------: |
| Full range  |       1x       |    Low    |
| ±10%        |     ~4.5x      |  Medium   |
| ±5%         |      ~9x       |   High    |
| ±1%         |      ~45x      | Very High |

The relationship isn't linear — it's closer to hyperbolic, which is why tight ranges can feel like playing with fire.

---

## Rebalancing Strategies

When price exits your range, you stop earning. You have a few options:

1. **Do nothing** — wait for price to return (hope is not a strategy)
2. **Manual rebalance** — close and reopen at the new price
3. **Automated rebalance** — use triggers to reposition programmatically

I've been building tools to backtest option 3 systematically. The two main strategies I work with:

### Percentage-based Rebalance

```python
def should_rebalance(current_price, range_lower, range_upper, threshold=0.02):
    """Trigger rebalance when price is within threshold of range edge."""
    range_width = range_upper - range_lower
    distance_to_lower = (current_price - range_lower) / range_width
    distance_to_upper = (range_upper - current_price) / range_width

    return min(distance_to_lower, distance_to_upper) < threshold
```

### Sequential Rebalance

This approach stacks multiple ranges and shifts the active set as price moves:

```javascript
const ranges = [
  { lower: 1800, upper: 2000, weight: 0.5 },
  { lower: 2000, upper: 2200, weight: 0.3 },
  { lower: 2200, upper: 2400, weight: 0.2 },
];

// When price crosses a boundary, rotate the set
function rotateRanges(currentPrice, ranges) {
  return ranges.map(r => ({
    ...r,
    active: currentPrice >= r.lower && currentPrice <= r.upper
  }));
}
```

---

## Backtesting Results

After running Monte Carlo simulations across ~10,000 price paths for ETH/USDC:

- Tight ranges (±2%) with **frequent rebalancing** outperform wide ranges in low-volatility regimes
- In high-volatility regimes, gas costs and slippage **eat most of the edge**
- The sweet spot tends to be around ±5-8% with rebalancing triggers at 70-80% range consumption

> **Key insight**: The optimal strategy isn't static. It shifts with volatility regime, gas costs, and pool depth. This is why backtesting matters.

### Checklist for a Good Backtest

- [x] Use real historical tick data, not OHLC
- [x] Account for gas costs per rebalance
- [x] Model slippage on entry/exit
- [x] Include fee compounding
- [ ] Factor in MEV (still working on this)
- [ ] Cross-pool arbitrage effects

---

## The Tools

I built two tools to make this analysis accessible:

**[clAMM Analyzer](/dextools/clamm_analyzer_1.7.html)** — Interactive position analyzer with multi-range support, PnL tracking, and fee projection. Paste a position ID or configure manually.

**[Rebalance Backtester](/dextools/clamm_backtester_1.1.html)** — Run historical backtests on rebalancing strategies with configurable triggers, Monte Carlo optimization, and exportable results.

Both are standalone HTML tools — no backend, no tracking, no wallet connection required.

---

## What's Next

A few things on the roadmap:

1. **Volatility regime detection** — automatically classify market conditions and suggest range parameters
2. **Multi-pool strategies** — distributing liquidity across correlated pairs
3. **Real-time monitoring** — webhook alerts when positions need attention

The goal is to make concentrated liquidity less of a guessing game and more of an engineering discipline.

---

*If you're working on similar problems or want to collaborate, find me on [X](https://x.com/ilo_0x).*
