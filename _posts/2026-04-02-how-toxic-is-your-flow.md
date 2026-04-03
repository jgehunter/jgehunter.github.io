---
layout: single
title: "How Toxic Is Your Flow?"
date: 2026-04-02
categories: [microstructure]
tags: [markout, adverse-selection, market-making, crypto, markoutlib]
toc: true
toc_sticky: true
mathjax: true
---

{% include figure image_path="https://static.news.bitcoin.com/wp-content/uploads/2026/02/war.png" caption="Source: [Bitcoin.com](https://news.bitcoin.com/middle-east-explosions-and-us-iran-military-escalation-rip-through-bitcoins-price-action/)" %}

At 05:30 UTC on February 28, 2026, the United States [launched strikes on Iran](https://www.centcom.mil/MEDIA/PRESS-RELEASES/Press-Release-View/Article/4418396/us-forces-launch-operation-epic-fury/). Within thirty minutes, [$300 million in leveraged crypto positions were liquidated](https://www.coindesk.com/markets/2026/02/28/bitcoin-slides-under-usd64-000-as-u-s-and-israel-launch-strikes-on-iran) and roughly $5 billion in BTC left the major exchanges, [erasing $128 billion in market cap](https://www.fxleaders.com/news/2026/02/28/bitcoin-price-braces-for-60k-as-us-iran-strikes-erase-128-billion-in-flash-crash/). If you were a market maker with live quotes on Binance at that moment, you potentially had a very bad morning.

But how bad, exactly? And, more usefully: how bad was it compared to the hours just before, when nothing in particular was happening?

This is what markout analysis can help with. A markout is the mid-price movement after a trade, measured from the maker's perspective. You fill a buy order, the mid rises, you lost money. The markout tells you how much and how fast. It's the most direct measure we have of adverse selection, and it's the number that every market-making desk tracks obsessively. All markouts in this post are computed on a mid-to-mid basis: the starting reference is the mid-price at the time of the fill, not the trade price itself. The alternative conventions (fill-to-mid, which starts from the trade price, and fill-to-cross, which measures trade price to trade price) answer slightly different questions about execution quality, but mid-to-mid isolates the pure information content of the trade from the spread the maker already earned.

What follows is a walkthrough of real Binance trade data from that day, and the day after. I want to understand three things: what a normal hour looks like for a BTC market maker, what happens when a geopolitical shock hits, and what the microstructure of DOGE looks like compared to BTC (spoiler: they are very different animals). Along the way we'll stumble into some uncomfortable questions about who was selling before the strike and whether they should have been.

A note on the data before we start: Binance aggTrades aggregate all fills at the same price level into a single record, so a single "trade" here may represent one large order or many small limit orders consumed by a sweep. The `is_buyer_maker` flag tells us who crossed the spread, but crossing the spread doesn't imply a directional view. A stop-loss triggering, an arb closing a basis, and a retail panic-sell all look the same. Keep that in mind.

All analysis uses [markoutlib](https://github.com/jgehunter/markoutlib), an open-source library I built for this kind of work. A [companion notebook](https://github.com/jgehunter/markoutlib/blob/main/examples/05_blog_how_toxic.ipynb) reproduces every figure and table in this post.

# A quiet hour and a loud one

```python
import polars as pl
import markoutlib as mo
from fetch_data import fetch_binance_aggtrades, prepare_binance_trades_and_quotes

raw = fetch_binance_aggtrades("BTCUSDT", "2026-02-28")
trades, quotes = prepare_binance_trades_and_quotes(raw)
```

I split the day into two windows: pre-strike (00:00 to 05:00 UTC, a quiet Asian session) and post-strike (05:30 to 11:00 UTC, the immediate aftermath). The pre-strike window has 119,000 trades. The post-strike window has 588,000, roughly five times the activity in a comparable span of time. That ratio alone tells you something.

```python
result_pre = mo.compute(
    trades=trades_pre, quotes=quotes_pre,
    horizons=mo.seconds(1, 5, 10, 30, 60, 300),
    unit="bps", perspective="maker",
)
result_post = mo.compute(
    trades=trades_post, quotes=quotes_post,
    horizons=mo.seconds(1, 5, 10, 30, 60, 300),
    unit="bps", perspective="maker",
)
```

![BTC markout curve: pre vs post Iran strike](/assets/images/markout-post/01_pre_post_curve.png)

During the quiet hours the maker loses about 0.7 bps in the first second and the curve flattens around 1 bps by 30 seconds, which is roughly the terminal adverse selection cost per fill. We can quantify how fast the curve reaches that terminal level by fitting an exponential decay.[^1] The fitted half-life is 0.6 seconds, meaning half the total adverse price movement happens within 0.6 seconds of the fill.

The half life gives us an idea of how persistant the information in each trade is. A short half-life means the mid adjusts and settles quickly. The information is a pulse, a one-off shock that the book absorbs. A long half-life means the mid keeps drifting in the same direction well after the fill. The information is a trend. This distinction matters enormously for how you manage inventory, because a pulse and a trend require completely different responses even if the instantaneous cost is the same.

Post-strike, the 1-second markout nearly doubles to -1.37 bps. But the real story is at longer horizons. Instead of flattening, the curve keeps deepening: -2.1 bps at 30 seconds, -2.5 at a minute, -3.3 at five minutes. The half-life has stretched from 0.6 seconds to 3.6 seconds. The regime has changed from pulse to trend. Each fill is no longer a one-off adverse event that the book absorbs; it's the start of a sustained drift that lasts minutes.

# Who's hurting you?

```python
trades_tagged = trades_post.with_columns(
    pl.when(pl.col("size") <= q33).then(pl.lit("small"))
    .when(pl.col("size") <= q67).then(pl.lit("medium"))
    .otherwise(pl.lit("large")).alias("size_bucket")
)
result_sized = mo.compute(
    trades=trades_tagged, quotes=quotes_post,
    horizons=mo.seconds(1, 5, 10, 30, 60, 300),
    unit="bps", perspective="maker",
)
result_sized.plot.curve(by="size_bucket")
```

![Post-strike markout by trade size](/assets/images/markout-post/02_size_buckets.png)

At 1 second the three size buckets are bunched together, -1.3 to -1.5 bps. But watch the curves diverge at longer horizons. By 5 minutes, large trades have separated decisively at -3.5 bps while small and medium sit around -2. The information in large fills is more persistent. It keeps pushing the mid long after the trade.

In calmer markets you'd expect the opposite. Informed participants typically split large orders into small clips to hide their intent, which is just standard algo behaviour. During a geopolitical shock, they stop bothering. When the US is striking Iran and BTC is in freefall, the cost of being wrong on timing dwarfs the cost of signalling. Large fills during a crisis become the clearest signature of informed urgency, and the maker's default playbook of treating all fills equally breaks down exactly when it matters most.

# What the maker actually earns (and loses)

The Huang-Stoll identity[^2] decomposes the effective half-spread[^5] into three components: adverse selection (price impact), inventory holding costs, and order processing costs. In practice, with public trade data, we can only measure the two-way version: realized half-spread (what the maker keeps after prices move) and price impact (what the market takes back). The realized half-spread absorbs both inventory and order processing into a single residual. This means that when we see a negative realized spread, we can't separate "the maker was adversely selected" from "the maker was bearing inventory risk." Keep that in mind.

```python
result_pre.spread_decomposition(horizon=mo.seconds(5))
result_post.spread_decomposition(horizon=mo.seconds(5))
```

![Spread decomposition: Huang-Stoll identity](/assets/images/markout-post/03_spread_decomp.png)

Look at the pre-strike panel first. The effective half-spread is 0.29 bps per fill. The price impact at 5 seconds is 0.90 bps. The realized half-spread is -0.61 bps. Negative. The maker was losing money on every fill *before the bombs dropped*. This is the quiet Asian session, nothing happening, and the spread still doesn't cover the cost of adverse selection. If you thought the crisis was the problem, the decomposition says otherwise: the business model was already underwater.

The crisis just made it worse. Post-strike, the effective half-spread widens to 0.49 bps (up 73%). Sounds like the maker adapted. But price impact widened to 1.62 bps (up 80%). More spread earned, even more lost. The chart makes it visceral: the green bars barely change between panels, the red bars nearly double, and the orange bars go from bad to worse.

So where does the money come from? Exchange rebates fill in part of the picture. Binance VIP makers earn roughly 0.5 bps per fill at the top tiers.[^3] Adding that to the realized half-spread:

| Horizon | Realized half-spread | + Rebate | = Net P&L     |
| ------- | -------------------- | -------- | ------------- |
| 1s      | -0.41 bps            | +0.50    | **+0.09 bps** |
| 5s      | -0.61 bps            | +0.50    | -0.11 bps     |
| 60s     | -0.84 bps            | +0.50    | -0.34 bps     |
| 300s    | -0.51 bps            | +0.50    | **-0.01 bps** |

The maker is profitable at exactly one horizon: one second, at +0.09 bps per fill. At a more typical rebate of 0.25 bps, even that window is negative. The entire profitability picture rests on which rebate tier you're on, which is why market makers fight so hard for VIP status and why exchange rebate programmes are such effective tools for attracting liquidity. Maggio, Liu, Rizova, and Wiley (2020) document the same sensitivity in U.S. equities using VWAP-basis markouts: exchange fee structures are not incidental to execution costs, they are load-bearing.[^6]

The 300-second row is interesting also. The markout at 60 seconds is -1.12 bps but at 300 seconds it recovers to -0.79 bps, a 0.33 bps mean reversion that brings the top-tier maker approximately to breakeven. Whether that reversion is reliable enough to build a strategy around, or just a statistical artefact of this particular session, is an open question. But it does suggest that the 60-second markout overstates the terminal adverse selection.

These numbers paint a bleak picture, and yet crypto market makers do exist and some of them are quite profitable. The markout analysis measures the *passive* cost of providing liquidity: post a quote, get filled, sit there. A real maker doesn't sit there. They skew their quotes after each fill, lowering the ask when they're long to attract offsetting flow, raising the bid when they're short. This inventory management captures mean reversion that doesn't show up in a per-fill markout, because the profit comes not from any single fill but from the *sequence* of fills that flattens the position. They hedge on correlated instruments: a BTC spot fill might be immediately offset with a perp trade, capturing the basis while neutralising directional risk. And they segment their flow, quoting tighter where the flow is less toxic and wider (or not at all) where it isn't. The markout tells you the cost of the raw material. The margin comes from what you do with it. Mackintosh (2020) makes a related point: markouts taken in isolation can mislead as a measure of execution quality, precisely because they capture the passive fill but not the portfolio-level response that follows.[^7]

# A different market: DOGE

Everything so far has been BTC. DOGE is a different animal: thinner, more retail-dominated, about 140,000 trades per day on March 1 versus BTC's 1.76 million.

Comparing them in wall-clock (seconds) is misleading. "Five seconds later" means something very different in a market with 20 trades per second versus one with 1.6. So we switch to *trade-clock*: how much does the mid move after the Nth subsequent trade? This normalises for activity rate and gives the per-fill cost directly.

```python
result_btc = mo.compute(
    trades=trades_btc, horizons=mo.trades(1, 5, 10, 25, 50, 100),
    unit="bps", perspective="maker",
)
result_doge = mo.compute(
    trades=trades_doge, horizons=mo.trades(1, 5, 10, 25, 50, 100),
    unit="bps", perspective="maker",
)
```

![Trade-clock markout: BTC vs DOGE](/assets/images/markout-post/04_btc_vs_doge.png)

At the 1-trade horizon, each DOGE fill costs the maker 0.35 bps. Each BTC fill costs 0.02 bps. **17 times worse** per interaction. But the shapes matter more than the levels. BTC's curve keeps climbing steadily out to 100 trades; information drips in across many fills, each one a small incremental revelation. DOGE does almost all its damage in the first 10 trades and flatlines. One informed DOGE order *is* the information event. In BTC, an informed order is a contribution to a longer conversation.

Now the counterintuitive part. DOGE's effective half-spread is 1.27 bps, more than four times BTC's 0.29. Applying the same P&L accounting in trade-clock:

| Trades forward | Realized half-spread | + Rebate | = Net P&L     |
| -------------- | -------------------- | -------- | ------------- |
| 1              | +0.92 bps            | +0.50    | **+1.42 bps** |
| 5              | +0.03 bps            | +0.50    | **+0.53 bps** |
| 10             | -0.44 bps            | +0.50    | +0.06 bps     |
| 50             | -0.90 bps            | +0.50    | -0.40 bps     |

The DOGE maker earns **+1.42 bps per fill** at the 1-trade horizon. The BTC maker earns +0.09. DOGE looked 17 times more toxic, but the wider spread more than compensates. The catch: there's no mean reversion. After 10 trades the maker is underwater and it keeps getting worse. No "hold and wait for the bounce" option. DOGE market-making has a wider profitability window per fill and a harder cliff when you fall off it.

# Information leakage

Markout analysis extends naturally to *negative* horizons. Instead of asking "where does the mid go after the trade?", we ask "where *was* the mid before the trade?" If the mid is already trending in the trade's direction before the fill arrives, that's information leakage.

```python
result = mo.compute(
    trades=trades_post, quotes=quotes_post,
    horizons=mo.seconds_range(-30, 30, 1),
    unit="bps", perspective="maker",
)
result.plot.curve()
```

![Information leakage: crossing-zero curve](/assets/images/markout-post/05_info_leakage.png)

Read this chart left to right. At -30 seconds, half a minute *before* the trade hits, the maker's markout is already +2.36 bps. The mid has been drifting against the maker for thirty seconds before the fill even arrives. At -10 seconds, +1.55 bps. At -1 second, +0.87 bps. Then the trade lands and the markout snaps to -1.37 bps at +1 second.


# Who was selling before the strike?

That leakage curve measures the seconds around each trade. We can ask a coarser and more pointed question: what was the directional composition of flow in the *hours* before 05:30?

Within hours of the strike, CoinDesk [reported](https://www.coindesk.com/markets/2026/02/28/suspected-insiders-make-over-usd1-2-million-on-polymarket-ahead-of-u-s-strike-on-iran) that six newly created Polymarket wallets had collectively made $1.2 million by betting on the strike before it happened. Blockchain analytics firm Bubblemaps found the wallets were funded within 24 hours of the attack and had no prior trading history. A [Harvard Law paper](https://corpgov.law.harvard.edu/2026/03/25/from-iran-to-taylor-swift-informed-trading-in-prediction-markets/) later screened 93,000 Polymarket markets and found that one account placed its first trade **seventy-one minutes before the news broke**, when markets indicated only a 17% probability of a strike. Seventy-one minutes before 05:30 is approximately 04:19 UTC. Keep that timestamp in mind.

So informed pre-event positioning existed on Polymarket. Can we see anything similar in the Binance spot microstructure? To answer that we need a baseline. I pulled the same 15-minute flow analysis for February 22, the previous Saturday, when nothing in particular was happening.

```python
# Compare flow composition: strike day vs baseline
for day in [date(2026, 2, 28), date(2026, 2, 22)]:
    for window in fifteen_minute_windows(2, 6):
        sell_fraction = (trades_w["side"] == -1).sum() / len(trades_w)
        net_flow = (trades_w["side"] * trades_w["size"] * trades_w["price"]).sum()
```

![Pre-strike flow: Feb 28 vs Feb 22 baseline](/assets/images/markout-post/06_pre_strike_flow.png)

The top panel shows the taker sell fraction in 15-minute windows from 02:00 to 06:00 UTC. On the baseline Saturday (blue), it oscillates around 50% with no particular trend. On strike day (red), a sustained sell bias begins around 03:30 and persists through the strike. The 04:00 to 04:15 window hits 60.9% sells; on the baseline the same window was 41.8% (net buying). The bottom panel tells the same story in dollar terms: the flow imbalance flips persistently negative on strike day from 03:30 onward, while the baseline flips between positive and negative with no trend.

For context, across all 96 fifteen-minute windows on the baseline Saturday, a sell fraction above 60% occurred in exactly 2 windows (2.1%). On strike day, it happened in 4 of the 16 windows between 02:00 and 06:00.

The markouts sharpen the picture:

| Window      | Day    | Sell % | Flow imbalance | 60-min taker markout  |
| ----------- | ------ | ------ | -------------- | --------------------- |
| 04:00–04:30 | Feb 28 | 59.2%  | -25.4%         | **+2.07 bps (t=3.9)** |
| 04:00–04:30 | Feb 22 | 51.4%  | +1.5%          | +0.54 bps (t=1.0)     |
| 05:15–05:30 | Feb 28 | 62.3%  | -33.2%         | **+83.1 bps (t=5.1)** |
| 05:15–05:30 | Feb 22 | 41.1%  | +7.2%          | n/a                   |

On the baseline, the 04:00 to 04:30 taker markout at 60 minutes is +0.54 bps with a t-stat under 1. Not significant, which is what you'd expect: random flow on a quiet Saturday. On strike day, the same window produces +2.07 bps with a t-stat of 3.9. Those sellers were directionally correct at a horizon that extends through the crash. And recall that 04:19 is when the Harvard paper places the first informed Polymarket bet.

I want to be careful with this. A sell fraction of 61% in one 15-minute window is unusual (top 2% on the baseline) but not extraordinary. BTC was already weak from the tariff shock earlier that week, and the sell bias could reflect bearish trend-following in the Asian session. The 60-minute markout is positive because the downtrend continued, which it might have done regardless of the strike.

But the combination is what the markout framework makes visible: not just that sellers were active, but that those sellers were directionally correct at timescales extending through an event that, as far as the public record goes, hadn't been announced yet. The baseline comparison shows this pattern doesn't appear on a normal Saturday. And the timing alignment with the Polymarket evidence (the first informed bet at 04:19, the flow imbalance peaking in the 04:00 to 04:15 window) is, at minimum, notable. 

# What a maker should do with all this

The instinct during a crisis is to pull quotes. The half-life jumped 6x, every fill is toxic, get out. But pulling quotes means earning nothing during the window with 5x the normal volume. The post-strike session had 588,000 trades. That's the most liquidity demand of the entire day, and if you're not quoting, it's revenue walking past you.

Another possible answer is to widen. If the adverse selection doubled, just widen the spread to compensate and capture the gap. At 588,000 fills, even a thin per-fill edge at a wide spread beats any calm session.

But widening creates its own problem. When you widen your quotes, you filter out the uninformed retail flow that was subsidising you. The takers still willing to cross a 2 bps spread during a crash are disproportionately the ones who know something. So widening can increase your per-fill adverse selection at the same time it increases your per-fill revenue. Whether the spread widens faster than the toxicity depends on the market's composition at that moment, and composition is the one thing you can't directly observe. This is the Glosten-Milgrom problem[^4] in its purest form, and taken to its logical extreme, it's how markets break down entirely: spreads widen until no one trades.

In practice, crypto during a crisis doesn't fully spiral because the urgency is broad enough. Retail panicking out of a DOGE position will pay 2 bps. An arb between spot and perp will cross any spread narrower than the basis. There's enough non-informed urgency to keep the adverse selection ratio from going to 100%. But *how much* is enough? That's the question, and it doesn't have a static answer.

What the data does show: the effective half-spread widened 73% post-strike while price impact widened 80%. The makers in the market were widening, but not fast enough. Whether the right move was to widen *more*, or to widen more *selectively* (quoting tighter for small orders and pulling for large, or quoting only on one side based on inventory), is the kind of question that separates the firms that make their year during five hours of crisis from the ones that just survive it.

The half-life is the closest thing to a real-time regime indicator I've found for this. At 0.6 seconds, information is a pulse. At 3.6 seconds, it's a trend. The shift tells you that the adverse selection you're absorbing has changed character, not just magnitude. A wider spread might cover the per-fill cost, but it won't help if the mid is trending persistently against your inventory for minutes at a time. That's not a spread problem. It's an inventory problem, and it needs a different toolkit: position limits, directional skew, hedging on a correlated instrument. Whether any of that is enough to avoid the adverse selection spiral depends on the market, the counterparties, and how many other makers just pulled their quotes. But at least the half-life gives you a number that may help you get a grip on which regime you are at.

[^1]: The half-life is estimated by fitting an exponential decay $m(h) = m_\infty(1 - e^{-h/\tau})$ to the markout curve. The half-life is $\tau \ln 2$.

[^2]: Huang, R.D. and Stoll, H.R. (1997), "The Components of the Bid-Ask Spread: A General Approach", *Review of Financial Studies*, 10(4), 995–1034.

[^3]: Binance VIP maker rebates vary by tier. 0.5 bps is representative for the highest tiers (VIP 8-9); most active makers are closer to 0.2–0.3 bps, at which point the 1-second window is also negative. The entire profitability picture is sensitive to this number.

[^4]: Glosten, L.R. and Milgrom, P.R. (1985), "Bid, Ask and Transaction Prices in a Specialist Market with Heterogeneously Informed Traders", *Journal of Financial Economics*, 14(1), 71–100.

[^5]: Throughout this post, "effective spread" refers to the effective half-spread: $\text{side} \times (P - M)$. The full effective spread in the Huang-Stoll sense is twice this value.

[^6]: Maggio, M. Di, Liu, J., Rizova, S., and Wiley, R. (2020), "Exchange Fees and Overall Trading Costs", *Working Paper*, Dimensional Fund Advisors. Available at [SSRN](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3625801).

[^7]: Mackintosh, P. (2020), "What Markouts Are and Why They Don't Always Matter", *Nasdaq Market Insights*.
