---
layout: single
title: "Pricing the Champions Cup final on Polymarket: a Bayesian rugby model that disagrees with me, the market, and itself"
date: 2026-05-18
categories: [prediction-markets]
tags: [bayesian, rugby, Champions Cup, Polymarket, Bordeaux, Leinster, market-making, glicko]
excerpt: "I'm driving to Bilbao to support Bordeaux Bègles in the 2026 European Rugby Champions Cup final. My model wants me to bet on Leinster. The story of how the model became sceptical, what it took to bring it round, and where it still won't budge."
toc: true
toc_sticky: true
mathjax: true
---

I'm driving to Bilbao on Friday. On Saturday at San Mamés, Leinster will play Bordeaux Bègles in the European Rugby Champions Cup final and I would like Bordeaux to win. They are the defending champions, the team that ended a generation of French frustration in this tournament, and they have come back for more. I have a ticket. I have a hotel. I have an opinion.

Polymarket has Bordeaux as a strong favourite at 60.5%. William Hill agree, at the equivalent of 77%. Coral at 75%. Paddy Power somewhere in between. The market has done its sums and decided that the team I want to win is the team that ought to win.

My model, after two weeks of being built, tested, broken, fixed, and tested again, thinks Bordeaux is a 45% chance. Leinster is the slight favourite.

So I have an awkward problem. I support a team that the market backs and the model fades. The post you are reading is what happens when you set yourself the analytical challenge of pricing a binary settlement market on a sports final, then your model insults the team you fly in to watch.

## Why Polymarket on a rugby final

I work in eFX market making. The day job is to quote prices in a market where the fundamental never reveals itself, only other people's prices do. Whether my quotes are good is something I have to infer from markout against future midpoints, which is roughly like grading exams by comparing each answer to the answer most popular among the rest of the class.

A prediction market on a sports final is the rare laboratory where this changes. The settlement is binary. The fundamental price reveals itself in finite time. Polymarket on Saturday at full-time becomes a 1 or a 0 forever, and the rest of the week is an unusually pure RFQ pricing exercise: there is a prior, there is incoming evidence from the order book, there is a posterior, and there is a quote.

If a quoting framework works on a Polymarket binary, you should at least consider taking it seriously elsewhere. If it doesn't work here, where ground truth lives 24 hours away, you should be suspicious of anyone who tells you their FX markout looks great. The whole point of building a rugby model is not the rugby. It is the architecture.

That said, the rugby matters, because the rugby is the prior.

## The framework, in one diagram

The setup is a Bayesian binary on the home-win event:

1. Build a generative model of European club rugby match outcomes from twelve years of data. Use it to derive a structural prior over `P(Bordeaux wins)`.
2. Treat the Polymarket midpoint as a noisy observation of the true probability, with a Gaussian likelihood on the logit scale and a calibrated noise variance `σ_m`.
3. Update the prior to a posterior, once a day, every day for the week before kickoff.
4. Quote a bid and an ask around the posterior using a market-making framework adapted from Avellaneda-Stoikov, with the inventory term decaying as time-to-settlement shrinks (the opposite of FX RFQ pricing, which is the actual interesting mathematical feature).

This post is mostly about the prior. The prior is where most of my time went, and where the model and I had the largest argument.

## The data

The training set is every European club rugby match across four competitions from 2014-15 through 2025-26. Champions Cup, Top 14, Premiership, and the URC (and its Pro12 / Pro14 predecessors). After patching in the 2025-26 European knockouts that the upstream source forgot to backfill, and pulling the Feb-May regular season directly from ESPN scoreboards, the panel is 5,604 matches with home and away scores, tries, cards, attendance, and venue. The data is alias-resolved across sources so that "Bordeaux", "Bordeaux-Begles", "Union Bordeaux Bègles", and "Bordeaux Bègles" all become the same team.

The headline thing the data wants to tell you is in the empirical try-distribution, before any model is fit:

![Tries distribution: observed vs independent Poisson]({{ '/assets/images/champions-cup-final/fig03_dc_signature.svg' | relative_url }})

Each cell shows how often a given home-tries / away-tries combination occurs relative to what an independent Poisson product would predict. Two facts jump off the heatmap. First, the (0, 0) cell is lifted 3.7 times above independence: defensive rugby matches are *much* more common than independent score distributions would predict. Second, the rest of the diagonal is gently elevated and the asymmetric corners (one team shut out while the other scores freely) are suppressed by about 40%. Defensive matches stay defensive on both sides. Attacking matches see both teams score. This is the textbook Dixon and Coles 1997 pattern, only stronger than the football literature typically reports.

So Dixon-Coles on tries is a defensible starting model, and I tried it. So is Glicko-2 on outcomes. So is a Bayesian DC via NUTS. So is a LightGBM on engineered features. The full ladder went through five point-estimate models and three additional Bayesian or ML variants before settling. The evaluation framework was always the same: walk-forward through the 2024-25 season, predict each match using only data strictly before it, score by Brier and log-loss and ECE.

Glicko-2 in fifty lines of NumPy beat everything else on the full holdout. The Dixon-Coles family clustered ~10% behind it. LightGBM with engineered features (form, head-to-head, cadence) tied Glicko exactly, meaning the rating differential is the only feature in our set that carries signal. This is the rare case where a 50-year-old rating system is the right model.

![Model ladder: Brier on 2024-25]({{ '/assets/images/champions-cup-final/fig04_model_ladder.svg' | relative_url }})

But Glicko-2 alone, with the home advantage tuned to 200 rating points on a 2022-24 validation window (rugby's home advantage is much bigger than the football literature would have you guess), gave a final-day prediction of `P(Bordeaux) ≈ 32%`. The market sat at 60%. A thirty percentage point gap on a market that big is uncomfortable to look at.

## A scepticism budget

When the model disagrees with the market by that much you have two honest options. You take the model at its word and you trade. Or you find a missing piece and you rebuild.

I tried the second, because thirty points is a lot to assert without having stress-tested. The criterion for any model change was the same throughout: does it improve Brier on the 2024-25 Champions Cup holdout matches by at least 3% relative, with no calibration degradation? Models that fail the criterion get documented and rejected, even (especially) when they would have pushed the prediction in a direction I'd have liked.

The hypotheses, in order tested:

**Squad rotation.** Leinster lost three URC matches early in 2025-26, including a 0-35 hammering by the Stormers in Cape Town. Were they rotated B-string sides on tour, and should those losses count less? I parsed lineups from the source, built per-team "A-teams" from Champions Cup starts (where everyone fields strong), and computed each match's rotation fraction. The Stormers and Bulls losses were indeed heavily rotated (0.53 fraction non-A-team). The Munster loss was full-strength. Building a rotation-weighted Glicko increases Leinster's effective rating slightly, *widens* the gap with the market by about 2 percentage points, and does not improve Brier on the cup holdout. Filed as a real diagnostic finding but does not earn a place in the model.

**Recency decay.** Glicko already favours recent matches implicitly through its RD shrinkage, but it does not have an explicit time decay on older matches. I built a variant with exponential weighting on a half-life of 60 to 1000 days. Every value of the decay parameter made the cup-holdout Brier worse than vanilla. Twelve-year-old matches are evidently still useful when fit through Glicko's mechanic. Rejected.

**Per-team cup-overperformance offsets.** Maybe certain teams elevate their game in Europe and the model should give them a rating bonus when predicting cup matches. I computed each team's residual win rate in cup matches relative to Glicko expectation, used it as a rating offset, and tested. Brier worsened uniformly across all scale parameters. Rejected.

**Style-aware ML.** Use LightGBM with engineered style features (rolling try-rate, rolling points conceded, head-to-head, days since last match). Brier on the cup holdout *got worse*, not better. The model's call on Saturday dropped Bordeaux from 39% to 30%. Rejected, with prejudice.

Four hypotheses tested, four rejected. The model and the market were starting to look genuinely at odds rather than tunable.

Then a more specific theory emerged: maybe Bordeaux has been pacing itself in Top 14 and concentrating on Europe, while Leinster's URC dominance is flattered by the relative weakness of their league. Two specific testable claims hidden inside that:

1. Bordeaux's domestic opposition is stronger than Leinster's domestic opposition.
2. Bordeaux's cup *margin of victory* is bigger than Leinster's, and Glicko ignores margin of victory entirely.

The strength-of-schedule diagnostic was decisive:

| | Bordeaux (Top 14) | Leinster (URC) |
|---|---|---|
| Mean opponent Glicko rating, 2025-26 | **1628** | **1523** |
| Mean point differential | +4.4 | +8.1 |
| Mean point differential in 2025-26 Champions Cup | **+19.0** | +14.0 |

Bordeaux's domestic opponents this season averaged 105 Glicko rating points stronger than Leinster's. Glicko-2 already adjusts for opponent rating in the per-match update, so this on its own is not a model flaw. But the next two rows are. **Bordeaux beat their Champions Cup opponents by an average margin of 19 points. Leinster beat theirs by 14.** Both teams won 7 of 7 on the way to the final. Bordeaux did it more emphatically. A 64-14 demolition of Leicester and a 30-15 takedown of Toulouse (Toulouse, top of the Top 14, the team Bordeaux's rating sits behind in my own model) are different from a one-point squeak past La Rochelle and a four-point grind past Toulon.

Glicko-2, as a standard rating system, throws all of that away. A win is a win whether by one point or fifty. That is fine when there is no other signal. When there *is* a signal as obvious as a 50-point spread between two finalists' cup margins, throwing it away is leaving money on the table.

## The change that earned its keep

I built a margin-of-victory variant of Glicko using the standard FiveThirtyEight-style adjustment: scale the per-match rating update by `ln(margin + 1) × 2.2 / (winner_rating_diff × 0.001 + 2.2)`. The autocorrelation term prevents stronger teams racking up rating points by running up the score against weak opposition, which is the obvious failure mode you want to design out.

The empirical result was unambiguous:

- **Brier on 2024-25 Champions Cup holdout: 0.1468 → 0.1396 (5% relative improvement)**
- **ECE on the same: 0.104 → 0.066 (better calibrated)**
- **Brier on the full 2024-25 holdout: 0.1715 → 0.1698 (no degradation outside cup matches)**

A clean win. The cup-only Brier passes the 3% graduation rule, the calibration improves, and the non-cup matches are not penalised. The change earns its keep on every metric I had decided in advance to look at.

When I plot per-team cup overperformance (mean cup margin minus Glicko-expected margin) it tells a story by itself:

![Cup point-margin overperformance vs Glicko expectation]({{ '/assets/images/champions-cup-final/fig09_cup_dominance.svg' | relative_url }})

Bordeaux is one of the top cup overperformers in the panel, at +4 points per match above their rating-implied margin. Leinster is not in the top ten. Edinburgh and Benetton, of all clubs, head the list, which is the kind of finding that earns the model some credibility: it is not just preferring the team I want to bet on.

And the proof point that I trust the most: the same model improvement, applied to the Challenge Cup final the night before, *brings the model's prediction into line with the bookmakers*. Montpellier are 75% favourites with the bookies. M2F gives them 75.1%. The Champions Cup adjustment fixed the Challenge Cup prediction as a side effect, which is exactly the signature you want from a real signal rather than a tune-to-target.

## The rating board, after all that

![Margin-aware Glicko ratings entering the 2026 final]({{ '/assets/images/champions-cup-final/fig05_rating_snapshot.svg' | relative_url }})

| | Rating |
|---|---|
| Leinster | 1947 |
| **Bordeaux Bègles** | **1914** |
| Toulouse | 1911 |
| Montpellier | 1867 |
| Northampton Saints | 1863 |
| Bath | 1861 |

Three teams within 36 rating points at the top. That is a much closer top tier than vanilla Glicko showed, where Leinster was 132 points clear of Bordeaux. Margin-of-victory pushed Bordeaux up faster than it pushed Leinster, because Bordeaux's Champions Cup wins were genuinely bigger.

And the headline:

![Market vs model on the 2026 final]({{ '/assets/images/champions-cup-final/fig01_market_vs_model.svg' | relative_url }})

| | P(Bordeaux wins) |
|---|---|
| Polymarket | 60.5% |
| William Hill | 76.9% |
| Coral | 75.0% |
| Vanilla Glicko (no MoV) | 39.2% |
| **M2F Margin-of-victory Glicko** | **45.3%** |
| M10 Bayesian Dixon-Coles | 33.1% (90% CI [28.2%, 38.7%]) |

The working model now sits at `P(Bordeaux) = 45.3%`, with Leinster the slight favourite. The disagreement with Polymarket has halved from 30 percentage points to 15. The disagreement with the regulated bookmakers is still substantial. The disagreement with Bayesian DC (which doesn't yet incorporate margin-of-victory and which therefore I'd weight lower) is now wider than the model-market gap.

The model and the market are converging, but not all the way. Polymarket thinks Bordeaux 60-40. The model thinks Bordeaux 45-55. We disagree by about a coin-flip-and-a-half on each side.

The honest list of what could still close the remaining gap:

1. **Bilbao is closer to Bordeaux than to Dublin.** The "neutral venue" flag in my data is a binary. There is a real chance the partisan crowd in Bilbao is worth a few percentage points to Bordeaux, and the model has none of that.
2. **The market sees team news, injuries, and a referee assignment that my model doesn't.** Polymarket has been trading this match for weeks. If a Leinster fly-half pulls out on Tuesday morning, the market knows by Tuesday afternoon and my prior doesn't.
3. **Recent-cup-form effects beyond margin of victory.** Bordeaux have looked, qualitatively, more dominant than my model captures. Whether the right way to encode "looked dominant" is more than a margin-of-victory adjustment, I don't know.
4. **The market is also wrong.** Markets are usually informed but not always right, and a 15-point disagreement is small enough to be plausibly explained by either side being marginally off.

I am not going to keep tuning. I tested eight different model improvements; one earned its keep; the others are documented as having been tested honestly and failed. The remaining gap will get closed over the week, not by my changing the model further, but by the Bayesian update sequentially incorporating market evidence.

## The match the night before

![Predicted try distribution: Bordeaux vs Leinster]({{ '/assets/images/champions-cup-final/fig07_predicted_score.svg' | relative_url }})

On Friday at 21:00 local, Ulster play Montpellier in the Challenge Cup final, on the same pitch. The bookmakers have Montpellier as a heavy favourite (~75% implied). My model has it at 75% Montpellier. The Glicko-2 ratings have Montpellier at 1867 and Ulster well below at around 1700. The model and the market agree completely on this one, which I take as evidence that the model is not systematically broken: in a match where the analytical answer is straightforward, it matches everyone else's analytical answer.

The two finals together make a nicer methodological exhibit than either alone. Same model, same data, same week. One prediction converges with the market. One stays 15 points apart. That asymmetry is the actual interesting object.

## What I do

Here is the awkward part.

I want Bordeaux to win. I am driving to Bilbao to watch them. I will spend Saturday afternoon in a yellow-and-black shirt, with a beer, hoping a team I respect carries the day for the second year running. I have a strong rooting interest in the model being wrong.

The model says Leinster, mildly. The market says Bordeaux, firmly. And the model is the artifact I have spent two weeks building and breaking and rebuilding precisely so that I'd trust it more than my instincts.

If I were trading this on Polymarket from a cold start I would absolutely fade the market. The model has been tested in every reasonable way I could think of, by metrics chosen in advance, on a holdout chosen in advance. The disagreement is robust. The only thing that has moved the prior in Bordeaux's direction was a principled change with an empirical Brier improvement on the cup holdout, and that change still has Leinster as the favourite.

So I will probably bet against the team I am driving four hundred kilometres to support. Small size. A sanity-check stake rather than a portfolio-management stake. Enough to carry some skin, but not enough to spoil the beer.

The whole thing I tell anyone who asks about market making and prediction markets is: the model has a job, and your gut has a different job, and you are not supposed to confuse them. If you've built a careful prior and the market is offering you a price outside your 90% credible interval, the framework is telling you something. The honest read of the math is to take the price.

Whether that survives kickoff on Saturday afternoon is a different question.

---

*See you in Bilbao.*
