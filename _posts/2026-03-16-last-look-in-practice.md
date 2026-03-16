---
layout: single
title: "Last Look in Practice"
date: 2026-03-16
categories: [microstructure]
tags: [fx, last-look, market-making, adverse-selection]
toc: true
toc_sticky: true
mathjax: true
---

I'm more and more interested these days in the *why's* of market microstructure. It's easy to lose sight of how path-dependent and inefficient some features of the market are. You'd hardly design them this way if you were dreaming up markets from scratch (unless you were trying to pad some particular person's pockets), but institutional inertia is a strong force.

Last Look is one of the big examples. In short: when a client sends a trade request to an FX dealer, the dealer reserves the right to reject that request if the market has moved against them during a brief window after receipt. That's it. A conditional veto on a trade the client thought was done.

To understand why this exists you have to think about life from the dealer's seat. FX is overwhelmingly OTC. There's no centralized public limit order book visible to all participants. Price formation is fragmented across bilateral dealer relationships and multi-dealer platforms, and because of that fragmentation, a dealer's quoted price can go stale before the client even manages to hit it. Every quote you put out there is a sitting duck, a free option for anyone fast enough to pick it off.[^1] And in a market where you might be streaming prices to hundreds of counterparties across multiple platforms simultaneously, the cumulative exposure from stale quotes is not trivial. Last Look emerged as the dealer's defence against this, a "market check" after receiving the order, rejecting if the underlying price has moved too far.

Now, the charitable framing (and I think it's more than just charity) is that Last Look lets dealers quote tighter spreads because they can filter out toxic flow after the fact. Without that safety net, a rational dealer would have to widen spreads to cover the expected cost of being picked off, and wider spreads hurt everyone. The less charitable framing is that it's an asymmetric mechanism where the dealer gets to back out of losing trades while the client is held to winning ones. The truth, as usual, is somewhere in between, and it depends entirely on *how* Last Look is implemented. Which brings us to the part where some people got a bit too creative.

In November 2015, the NYDFS fined Barclays $150 million for how it ran Last Look on its BARX platform from 2009 to 2014.[^2] The gist: Barclays applied an artificial hold time and had an asymmetric rejection rule. Orders where the price moved against the bank were flagged as "toxic" and rejected, while orders that moved in the bank's favour sailed through. That alone might have been survivable, but the internal culture sealed it. A Managing Director on the eFX desk instructed colleagues via email to "avoid mentioning the existence of the whole BATS Last Look functionality" and to "obfuscate and stonewall" if asked. When clients complained about rejection rates, they were told it was IT issues. After the investigation began, Barclays moved to symmetric rejection in late 2014 and eventually removed additional hold times entirely.[^3] The fine brought Barclays' total FX-related NYDFS penalties to $635 million, the Global Head of eFICC was terminated, and the case became the canonical example of Last Look abuse, less because the mechanics were unique (they weren't) and more because the emails were so spectacularly incriminating.

# The FX Global Code and the Push for Symmetry

The Barclays case, and the general whiff of sharp practice around Last Look, provided the backdrop for the FX Global Code[^4], a set of voluntary principles published in 2017 by the Global Foreign Exchange Committee (GFXC), a forum of central banks and private sector participants.

Principle 17 of the Code addresses Last Look directly. It establishes that Last Look should be used only for price and validity checks, not as a profit filter, not for information gathering, and not as an opportunity to trade ahead of the client's order. The Code doesn't mandate symmetric rejection, but it requires disclosure: LPs must tell clients whether their rejection logic is symmetric or asymmetric, the expected processing time, and how information from trade requests is handled during the hold window.

The GFXC followed up in August 2021 with a dedicated guidance paper on Last Look[^5]. The paper reinforced that the hold window should contain no additional delay beyond what's needed for the price and validity check. It also laid out what monitoring liquidity consumers should perform: tracking acceptance/rejection rates, comparing round-trip times for accepts vs. rejects, and monitoring whether the LP is trading during the hold window.

Colin Lambert, who has been beating this drum at The Full FX for years, reported in September 2023 that nine of the top 20 LPs on platforms had average reject times more than double their accept times, suggesting an additional hold time is being applied on rejects. One LP in the top 10 had a reject hold time more than *thirteen times* its accept hold time.[^8] Only four of the top 20 were anything like symmetric. The Code is voluntary, and roughly half the market appears comfortable treating it that way.

From the dealer's perspective, this is frustrating in a different way than you might expect. If you're a desk that has invested in making your Last Look genuinely symmetric, with zero additional hold time, you're competing against LPs who are gaming the window to cherry-pick flow. They can quote tighter because they're rejecting more, and the client, looking at a screen full of indicative prices, has no easy way to distinguish between an honest tight spread and a tight spread that comes with a 30% rejection rate. The honest dealer ends up wider on the screen and lower on the ladder. Lambert's data suggests that the handful of LPs actually playing by the rules aren't being rewarded for it, which is not exactly a great incentive structure.

# How to Reason About Last Look

There's more than one way to conceptualize this, but one framework I've found particularly useful is to think in terms of options. A dealer's standing offer to trade is, economically speaking, a free option granted to the client. That option is live from the moment the quote is sent until it's cancelled or updated. And like any option, its value scales with two things: volatility and time to expiry, the familiar σ√t. The more volatile the market and the longer the quote sits out there, the more valuable the option the dealer is giving away for free.

Last Look is, in this framing, the dealer clawing back some of that unpaid option premium. When the client hits the quote and the dealer holds the request for some window τ, the dealer is effectively observing whether the option has gone in-the-money beyond some threshold. If it has, reject. If it hasn't, fill.

This is exactly the framing Roel Oomen adopted in his comprehensive treatment of the problem[^9], which analyses symmetric and asymmetric Last Look designs within a unified framework and shows how different protocols map to different effective transaction costs for the client. Álvaro Cartea, Sebastian Jaimungal and Jamie Walton take this further in a multi-venue equilibrium model[^10], showing that a zero-profit broker sets spreads that depend on the rejection threshold, and that, counterintuitively, a Last Look venue can sometimes quote *wider* spreads than a firm-quote venue. NBIM (the Norwegian sovereign wealth fund, so not exactly a disinterested party, being one of the world's largest buy-side FX participants) also published a clear practitioner-oriented perspective paper[^11] framing Last Look explicitly as an option payoff with knockout barriers.

This creates a fundamental asymmetry: the client only exercises the "option" (sends a trade request) when the price is favourable to them, and the dealer only rejects when it's unfavourable to the dealer. Both sides are selecting, and the client's fill distribution ends up truncated. They systematically get filled on the trades that didn't move much and rejected on the ones that did.

For the latest academic treatment, Alexander Barzykin (at HSBC, and one of the more productive practitioner-academics in this space) published a paper just this month that models the rejection decision as an explicit control variable jointly optimised with quotes[^12]. Crucially, he introduces a reputation mechanism where past rejections endogenously reduce future client flow, a formalisation of something every eFX desk already knows: reject too much and the flow dries up. It's built on the broader FX market-making framework he developed with Philippe Bergault and Olivier Guéant[^13], and their recent work on adverse selection and price reading[^14] is also worth a look if you're in this space.

# Mathematical Framework

Let's put some structure on the intuition above. The goal isn't a full equilibrium model. It's to make the options analogy precise enough to reason about quantitatively, and then to see what happens when we throw realistic numbers at it.

## Setup

Suppose a dealer quotes a mid price $$M_0$$ at time $$t = 0$$ with a half-spread $$s$$, so the client can buy at $$M_0 + s$$ (the ask). A client hits the ask at $$t = 0$$, and the dealer holds the request for a Last Look window of duration $$\tau$$.

During that window, the mid price evolves as a Brownian motion:

$$M_\tau = M_0 + \sigma W_\tau$$

where $$\sigma$$ is the instantaneous volatility of the pair and $$W_\tau \sim \mathcal{N}(0, \tau)$$, so the price change over the window is $$\Delta M = M_\tau - M_0 \sim \mathcal{N}(0, \sigma^2 \tau)$$.

## Rejection Rule

The dealer rejects if the mid has moved against them by more than a threshold $$\theta$$. For a client buy, the dealer is short after the fill, so adverse movement means the mid rising. The fill condition is:

$$\Delta M \leq \theta$$

That is, the trade goes through only if the mid hasn't risen by more than $$\theta$$ during the hold window. For $$\theta = 0$$, this is "reject if any adverse movement at all," the most aggressive possible Last Look. For $$\theta \to \infty$$, this is no Last Look at all.

Under an **asymmetric** design (the pre-Barclays default for many LPs), the dealer only rejects adverse moves. Under a **symmetric** design (now the expected standard per the Global Code), the dealer also rejects if the price moves in the dealer's favour beyond $$\theta$$, i.e. if $$\lvert\Delta M\rvert > \theta$$. The symmetric design means the dealer is also giving up trades where the client would have been filled at a price that turned out to be favourable to the dealer.

## Fill Probability

For the asymmetric case, since $$\Delta M \sim \mathcal{N}(0, \sigma^2 \tau)$$:

$$P(\text{fill}) = P(\Delta M \leq \theta) = \Phi\!\left(\frac{\theta}{\sigma\sqrt{\tau}}\right)$$

For the symmetric case:

$$P(\text{fill}) = P(\lvert\Delta M\rvert \leq \theta) = 2\Phi\!\left(\frac{\theta}{\sigma\sqrt{\tau}}\right) - 1$$

where $$\Phi$$ is the standard normal CDF. Both depend on $$\theta$$ only through the ratio $$\theta / \sigma\sqrt{\tau}$$. Doubling the threshold has the same effect as quartering the window duration (at fixed vol), or halving volatility (at fixed window). It's all one knob.

## Dealer's Expected P&L Per Filled Trade

Conditional on filling the client buy at the ask $$M_0 + s$$, the dealer's instantaneous P&L is:

$$\text{P\&L} = (M_0 + s) - M_\tau = s - \Delta M$$

Under the **asymmetric** design, we only fill when $$\Delta M \leq \theta$$, so the expected P&L per *filled* trade is:

$$\mathbb{E}[\text{P\&L} \mid \text{fill}] = s - \mathbb{E}[\Delta M \mid \Delta M \leq \theta]$$

The conditional expectation of a truncated normal is a standard result:

$$\mathbb{E}[\Delta M \mid \Delta M \leq \theta] = -\sigma\sqrt{\tau} \cdot \frac{\phi(z)}{\Phi(z)}$$

where $$z = \theta / \sigma\sqrt{\tau}$$ and $$\phi$$ is the standard normal PDF. That ratio $$\phi(z)/\Phi(z)$$ is the inverse Mills ratio. So:

$$\mathbb{E}[\text{P\&L} \mid \text{fill}] = s + \sigma\sqrt{\tau} \cdot \frac{\phi(z)}{\Phi(z)}$$

The second term is always positive. Last Look systematically improves the dealer's expected P&L on filled trades by filtering out the worst adverse moves. The dealer captures the spread *plus* a bonus from truncation.

Under the **symmetric** design, the conditional expectation $$\mathbb{E}[\Delta M \mid \lvert\Delta M\rvert \leq \theta] = 0$$ by symmetry, so the dealer's expected P&L per filled trade is simply $$s$$, the half-spread with no truncation bonus. This is a clean result, but it relies on the assumption that $$\Delta M$$ is centred at zero. In practice, it isn't.

## Adding Adverse Selection

The model so far assumes that the price change over the hold window is centred at zero, that conditional on a trade request arriving, the price is equally likely to move in either direction. If you've spent any time on an eFX desk, you know this is nonsense. The very fact that a client has chosen to hit your ask is informative. It tells you the price is, on average, more likely to move up than down. Clients with better information, faster feeds, or more sophisticated execution logic will disproportionately hit quotes that are about to become stale. This is adverse selection, and it's the central fact of life in streaming market making.

To capture this, we introduce an adverse selection parameter $$\alpha > 0$$ representing the expected drift in the price change conditional on a trade request arriving. For a client buy:

$$\Delta M \sim \mathcal{N}(\alpha, \sigma^2 \tau)$$

The parameter $$\alpha$$ is the expected adverse move the dealer faces *per trade request*. In practice, $$\alpha$$ varies by client tier (informed vs. uninformed), by time of day (more adverse selection around data releases), and by pair (more in liquid G10, less in EM). Empirically, for EUR/USD on MDPs, $$\alpha$$ might range from 0.05 pips for a corporate hedger to 0.3+ pips for an aggressive systematic flow.

This changes everything about the symmetric case. With the biased distribution, even symmetric rejection around $$\lvert\Delta M\rvert > \theta$$ filters out more adverse moves than favourable ones, because the distribution is shifted toward positive $$\Delta M$$ (bad for the dealer on a client buy). The conditional expectation given a symmetric fill is now:

$$\mathbb{E}[\Delta M \mid \lvert\Delta M\rvert \leq \theta] > 0$$

but it's *less than* $$\alpha$$, the unconditional adverse selection. The dealer still benefits from symmetric rejection because truncation pulls the conditional mean back toward zero. The dealer's expected P&L per filled trade under symmetric rejection becomes:

$$\mathbb{E}[\text{P\&L} \mid \text{fill}] = s - \mathbb{E}[\Delta M \mid \lvert\Delta M\rvert \leq \theta]$$

This is less than $$s$$ (unlike the zero-drift case), but greater than $$s - \alpha$$ (what the dealer would earn with no Last Look at all). Symmetric Last Look doesn't eliminate adverse selection, but it mitigates it. The truncation clips the tails of the biased distribution, and because the adverse tail is fatter than the favourable tail, the clipping is asymmetric in its effect even though the rule is symmetric in its design.

The chart below makes this point more clearly than algebra can. We plot $$\mathbb{E}[\Delta M \mid \text{fill}]$$ (the "shadow spread" the dealer faces on filled trades) as a function of the hold window $$\tau$$, for both designs and several levels of adverse selection, with parameters calibrated to EUR/USD ($$\sigma$$ = 8% annualised, $$\theta$$ = 0.10 pips, half-spread = 0.3 pips).

![Shadow Spread by Design and Adverse Selection](/assets/images/last-look/chart2_shadow_spread.png)

The dashed blue line is the textbook case everyone focuses on: symmetric rejection with no adverse selection, which gives you a shadow spread of exactly zero. Nice in theory. But add adverse selection ($$\alpha$$ = 0.15 or 0.30, the solid blue and orange lines) and the shadow spread goes positive and grows with the hold window. The red line shows asymmetric rejection without adverse selection, which *also* generates a substantial shadow spread, but in the opposite direction: the dealer actually benefits from truncation even on uninformed flow. This is the extraction the Global Code is designed to prevent.

The crucial observation is that the blue lines sit between the red line and the dashed line. Symmetric rejection with adverse selection gives the dealer real protection (the gap between the blue line and the dashed line) without the indiscriminate extraction of the asymmetric design. The more toxic the flow (higher $$\alpha$$), the more valuable symmetric rejection becomes.

This is the key insight that makes symmetric Last Look defensible from the dealer's perspective, not just the client's. Without the adverse selection bias, symmetric rejection would be a pure stale-quote filter with no P&L impact, a nice thing to put in your disclosure documents but not something that actually changes your economics. With the bias, it's a meaningful risk management tool that reduces the dealer's expected loss per trade without introducing the asymmetric extraction that the Global Code is designed to prevent. You're not clawing back option premium from uninformed corporates. You're protecting against the informed fraction of your flow.

## The Client's Perspective

Under the asymmetric design with adverse selection, the client's expected cost per filled trade is:

$$\mathbb{E}[\text{cost} \mid \text{fill}] = s - \mathbb{E}[\Delta M \mid \Delta M \leq \theta]$$

where the conditional expectation accounts for both the adverse selection drift and the truncation. This is strictly greater than $$s$$, the nominal spread. The excess over $$s$$ is the **shadow spread**: a hidden cost the client pays on top of the quoted spread, arising from the combined effect of adverse selection and asymmetric truncation. But the client also bears the cost of *rejected* trades: when a trade is rejected, the client re-enters the market at a worse price, or misses the opportunity entirely. The total effective cost must account for both the shadow spread on fills and the opportunity cost of rejects. Cartea, Jaimungal & Walton give the formal treatment[^10].

Under the symmetric design with adverse selection, the shadow spread is smaller but *does not vanish*. It's positive because the truncation acts on a biased distribution. The practical difference is that the symmetric shadow spread arises from genuine adverse selection mitigation rather than from indiscriminate extraction, which is a qualitatively different (and more defensible) economic story.

## What the Knobs Do

This gives you a clean way to reason about the three parameters: $$\sigma$$, $$\tau$$, and $$\theta$$.

- **Longer hold window** ($$\tau$$ ↑): More price information for the dealer to act on, higher rejection rates, larger shadow spread. From the dealer's seat, a longer window gives you more protection against toxic flow, but it also gives clients more reason to route elsewhere, and it gives Lambert more ammunition for his column. From the client's seat, the free option you've implicitly sold the dealer is now worth more, and Last Look is clawing more of it back.
- **Tighter threshold** ($$\theta$$ ↓): More aggressive rejection, same directional effect. At $$\theta = 0$$ under asymmetric rejection, the dealer rejects any adverse tick at all. Fill rates drop to 50% and the shadow spread blows up. Even under symmetric rejection, very tight thresholds produce high reject rates, which creates its own problems: client complaints, lower ladder positioning, and potential Code compliance issues.
- **Higher volatility** ($$\sigma$$ ↑): The option is worth more, so more gets clawed back. For the dealer, high-vol sessions are where Last Look earns its keep. The exposure from stale quotes scales with volatility, and without rejection the expected adverse selection cost per trade can exceed the half-spread. For the client, this means Last Look is most costly exactly when execution certainty matters most.

The uncomfortable conclusion, from whichever side of the trade you're sitting on, is that the client's *effective* spread is not what's on the screen. It's the quoted spread plus a volatility- and regime-dependent phantom cost that only shows up in fill statistics. And the dealer's *effective* economics depend not just on their quoted spread but on the interaction between their rejection policy, the toxicity of their flow, and the reputation consequences of rejecting too much.

## The Throughput Problem

Everything above is conditioned on fills. But a dealer doesn't just care about P&L per filled trade. You also care about how many trades you actually fill. A rejection policy that gives you great economics per fill but kills your fill rate might leave you worse off in aggregate. The right objective function for calibrating your threshold isn't $$\mathbb{E}[\text{PnL} \mid \text{fill}]$$ but $$\mathbb{E}[\text{PnL per request}] = \text{fill rate} \times \mathbb{E}[\text{PnL} \mid \text{fill}]$$, which penalises for throughput loss.

This reframing changes the picture substantially. The chart below shows per-request P&L as a function of adverse selection $$\alpha$$ for both designs at three different thresholds ($$\tau$$ = 75ms, calibrated to EUR/USD).

![Per-Request P&L vs Adverse Selection](/assets/images/last-look/chart6_pnl_per_request.png)

A few things jump out. First, the dashed black line (no Last Look) is a simple declining line: $$s - \alpha$$. No rejection, no fill rate penalty, but you eat the full adverse selection. Second, for benign flow (low $$\alpha$$), asymmetric Last Look actually *underperforms* no Last Look on a per-request basis, because the gain per fill doesn't compensate for the volume you're throwing away. Third, the shaded region between the no-LL line and the symmetric LL curves represents the net benefit of symmetric rejection. It grows with $$\alpha$$, confirming what we saw in the shadow spread chart: symmetric rejection is most valuable precisely when adverse selection is highest. And fourth, the threshold calibration matters. At $$\theta$$ = 0.05 pips (most aggressive), you're rejecting almost everything. At $$\theta$$ = 0.20 pips (loosest), you barely reject at all. The right threshold depends on your flow composition, which changes through the day and across client tiers.

This is what a desk actually calibrates to, whether they think about it in these terms or not. You want the threshold that maximises your expected revenue per unit of flow, accounting for the fact that flow is finite and rejection has both a direct cost (lost fills) and an indirect cost (clients routing elsewhere). Barzykin's recent paper[^12] formalises that indirect cost through an explicit reputation mechanism, and his optimal control solution for $$\theta^*$$ is considerably more nuanced than the static analysis here, but the intuition is the same.

# Conclusions

Last Look is one of those features of market structure that sounds reasonable when you hear the elevator pitch (protection against latency arbitrage, enabling tighter spreads) and gets progressively more interesting the deeper you dig.

The options framework is clarifying: every standing quote is a free option, and Last Look is the mechanism by which dealers claw back the unpaid premium. Under asymmetric rejection, clients pay a shadow spread on top of the quoted spread, scaling as $$\sigma\sqrt{\tau}$$ multiplied by the inverse Mills ratio. Under symmetric rejection *without* adverse selection, the shadow spread vanishes. But this zero-drift case is a theoretical convenience. In practice, the arrival of a trade request is itself informative, and the conditional distribution of price changes is biased against the dealer. Once you account for this adverse selection, symmetric rejection is no longer neutral: it still benefits the dealer by truncating the tails of a biased distribution, filtering more adverse moves than favourable ones even though the threshold is applied equally in both directions.

This is, I think, the strongest argument *for* symmetric Last Look. It's a genuine risk management tool against informed flow, not just a stale-quote filter, and it achieves this without the asymmetric extraction that the Barclays case made famous and that the Global Code is designed to prevent. It's a defensible position for a dealer to say "we apply a symmetric price check with zero additional hold time." It is not a defensible position to say "we reject everything that would have lost us money and fill everything that wouldn't," which is what asymmetric rejection amounts to in practice.

The FX Global Code, and particularly Principle 17 with its 2021 guidance paper, has pushed the market toward this standard. But "pushed toward" is doing a lot of work in that sentence. Lambert's data is not encouraging. The Code is voluntary, and voluntary codes work only to the extent that participants choose to follow them, or to the extent that clients choose to monitor and penalise non-compliance.

For a dealer running an honest book, the frustration is that Last Look compliance is not a visible competitive advantage. Clients see quoted spreads, not effective spreads. A desk that invests in symmetric rejection and fast processing gets punished on the screen by competitors who are gaming the window. The right response is more transparency, better TCA tools on the buy side, and (perhaps) moving the GFXC's guidance from a separate document into the Code proper, where it's harder to ignore.

For a client evaluating LP quality, the practical takeaway is this: the quoted spread is not your cost. Your effective cost is the quoted spread, plus the shadow spread from rejection, plus the opportunity cost of rejected trades, minus any price improvement passed through. Fill rate alone is a misleading metric. A 95% fill rate with asymmetric rejection can be more expensive than an 85% fill rate with symmetric rejection and tighter quoted spreads. The right diagnostic is to compare markout-adjusted fill cost, stratified by volatility regime, and to compare round-trip times for accepts vs. rejects across your LP panel. If your rejects are taking 13x longer than your accepts, that LP is not running a price check. They're running a P&L check.

The intellectual arc here is worth stepping back to appreciate. A practice that started as informal courtesy in voice-brokered FX ("let me check the market before I confirm") became encoded into electronic trading platforms, got systematically exploited, drew regulatory sanction, prompted a global code of conduct, and is now the subject of a rich academic literature connecting it to optimal control theory, option pricing, and mechanism design. Whether it survives in its current form, or whether the market gradually migrates toward firm-quote models on ECNs, will depend on whether the transparency the Code demands actually materialises, or whether institutional inertia, as usual, proves the stronger force.

---

*All views expressed here are my own and do not represent the views of my employer.*

---

[^1]: For a broader treatment of FX execution microstructure, including aggregation and latency effects, see Oomen, R. (2017). "Execution in an Aggregator." *Quantitative Finance*, 17(3), 383-404. [DOI](https://doi.org/10.1080/14697688.2016.1262538). See also Schrimpf, A. & Sushko, V. (2019). "FX trade execution: complex and highly fragmented." *BIS Quarterly Review*, December. [BIS](https://www.bis.org/publ/qtrpdf/r_qt1912f.htm).

[^2]: NYDFS (2015). "Consent Order: Barclays Bank PLC." [Full text (PDF)](https://www.dfs.ny.gov/system/files/documents/2020/04/ea151117_barclays.pdf). See also the NYDFS [press release](https://www.dfs.ny.gov/reports_and_publications/press_releases/pr1511181).

[^3]: By 2022, Barclays had formally removed additional hold times from its Last Look process. See FX Markets, June 2022: [Barclays scraps additional hold time on last look](https://www.fx-markets.com/trading/7946886/barclays-scraps-additional-hold-time-on-last-look).

[^4]: Global Foreign Exchange Committee (2024). "FX Global Code: A set of global principles of good practice in the foreign exchange market." [Full text (PDF)](https://www.globalfxc.org/uploads/fx_global.pdf).

[^5]: Global Foreign Exchange Committee (2021). "Guidance Paper on Last Look." [Full text (PDF)](https://www.globalfxc.org/uploads/gfxc_report_last_look-1.pdf). Developed by a working group led by then-GFXC co-Vice Chair Akira Hoshino.

[^8]: Lambert, C. (2023). "The Last Look..." *The Full FX*, 5 September 2023. [Full article](https://thefullfx.com/the-last-look-124/). Lambert has been tracking accept vs. reject round-trip times across top LPs on platforms and reporting the results publicly.

[^9]: Oomen, R. (2017). "Last Look." *Quantitative Finance*, 17(7), 1057-1070. [DOI](https://doi.org/10.1080/14697688.2016.1262545). Oomen was then global co-head of electronic FX spot trading at Deutsche Bank.

[^10]: Cartea, A., Jaimungal, S. & Walton, J. (2019). "Foreign Exchange Markets with Last Look." *Mathematics and Financial Economics*, 13(1), 1-30. [arXiv:1806.04460](https://arxiv.org/abs/1806.04460).

[^11]: NBIM (2015). "The Role of Last Look in Foreign Exchange Markets." *Asset Manager Perspective Series*. [Full text (PDF)](https://www.nbim.no/contentassets/bab2624ad58c4aa4aca65d19bfff2152/nbim_asset-managerperspective_3-15.pdf).

[^12]: Barzykin, A. (2026). "Dynamic slippage control and rejection feedback in spot FX market making." [arXiv:2603.07752](https://arxiv.org/abs/2603.07752). Published March 2026.

[^13]: Barzykin, A., Bergault, P. & Guéant, O. (2023). "Algorithmic market making in dealer markets with hedging and market impact." *Mathematical Finance*, 33(1), 41-79. [DOI](https://doi.org/10.1111/mafi.12373). This and their follow-up multi-currency paper in Risk.net form the backbone of the HSBC FX Research Initiative's market-making framework.

[^14]: Barzykin, A., Bergault, P., Guéant, O. & Lemmel, M. (2025). "Optimal Quoting under Adverse Selection and Price Reading." [arXiv:2508.20225](https://arxiv.org/abs/2508.20225).
