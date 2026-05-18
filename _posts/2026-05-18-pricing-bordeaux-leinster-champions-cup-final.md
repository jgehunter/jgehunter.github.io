---
layout: single
title: "Pricing the Champions Cup final on Polymarket: a Bayesian rugby model that disagrees with the market (and with me)"
date: 2026-05-18
categories: [prediction-markets]
tags: [bayesian, rugby, Champions Cup, Polymarket, Bordeaux, Leinster, glicko]
excerpt: "I'm driving to Bilbao to support Bordeaux Bègles in the 2026 European Rugby Champions Cup final. My model says I should bet on Leinster. Here's how it got there."
header:
  image: /assets/images/champions-cup-final/header_stade_chaban.jpg
  image_description: "Stade Chaban-Delmas, Union Bordeaux Bègles' home ground, before a rugby match"
  caption: "Stade Chaban-Delmas before a UBB match (photo: [R4gn0r0ck](https://commons.wikimedia.org/wiki/File:Stade_Chaban-Delmas_Rugby.jpg), [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/))"
  teaser: /assets/images/champions-cup-final/header_stade_chaban.jpg
toc: true
toc_sticky: true
mathjax: true
---

I'm driving to Bilbao on Friday. On Saturday at San Mamés, Leinster will play Bordeaux Bègles in the European Rugby Champions Cup final and I would like Bordeaux to win. They are the defending champions. I have a ticket. I have a hotel. I have an opinion.

Polymarket has Bordeaux at 60.5%. William Hill has them at the equivalent of 77%. Coral at 75%. Paddy Power somewhere in between. The market has done its sums and decided that the team I want to win is the team that ought to win, even if it can't quite agree by how much.

After two weeks of being built, tested, broken, fixed, and tested again, my model thinks Bordeaux is a 45% chance. Leinster is the slight favourite.

So I have an awkward problem. I support a team that the market backs and the model fades. What follows is what happened when I set myself the analytical challenge of pricing a binary settlement market on a sports final, and the model started arguing with the team I'm flying in to watch.

{% include figure image_path="/assets/images/champions-cup-final/fig00_hero.png" alt="Probability that Bordeaux Bègles wins: bookmakers, Polymarket, my model" %}

## Why the bookmakers and Polymarket don't agree

Before the model, a digression on the prices on the table. The three numbers I started with are 60%, 75%, and 77%, all on the same question. They are different for honest structural reasons rather than because anyone is being silly.

Bookmakers like William Hill and Coral set prices that include a margin to the house, and they price *defensively* on big public events: most punters who back this kind of final are betting on the favourite, the bookmaker shortens the favourite to protect itself, and the implied probability looks bigger than the bookmaker's actual fair-value estimate. The 77% and 75% lines are best read as "the bookmaker will not take a serious position the other way."

Polymarket is a peer-to-peer order book. There is no house, just the spread between the bids and offers that traders are willing to leave standing. The price reflects what the marginal trader is actually willing to risk capital on, and on a relatively low-liquidity market like a rugby final it sits closer to a genuine consensus view. Crucially Polymarket is what I can actually trade. The bookmakers' implied probabilities are useful context; Polymarket's number is the one I have to take a position against.

So when I talk about "the market" I mean 60.5%.

## What I'm doing

The setup is a simple Bayesian one. **Bayesian** here just means: I bring my own opinion to the question, the market brings its opinion, and I combine them into a posterior view, weighted by how much I trust each side.

1. Build a model of European rugby match outcomes from twelve years of data, and use it to estimate `P(Bordeaux wins)`. This is the **prior**: what I would say if I'd never seen the Polymarket price.
2. Treat Polymarket's price as a noisy reading of the true probability. Over the week, as new prices print, update my view toward the market in proportion to how much it has moved.
3. With a posterior on `P(Bordeaux wins)`, decide whether the live Polymarket offer is good enough to take. If the market is offering me Bordeaux at 60.5% and I think the true probability is 45%, the trade is to sell Bordeaux. If my posterior moves to 58% over the week, the offer isn't worth taking any more.

The post you're reading is mostly about the prior. The prior is where most of my time went, and where the model and I had the largest argument.

## Step one: what does the data actually look like?

Before fitting any model, I wanted to know what the empirical distribution of rugby outcomes looks like. The natural place to start is the **try distribution**: how often does a given (home tries, away tries) combination occur? Are home tries and away tries roughly independent of each other, or do they cluster?

The reason this matters: a venerable football model called **Dixon-Coles** (Dixon and Coles 1997) noticed that football scores aren't independent across teams. Specifically, more 0-0 and 1-1 matches happen than independent Poissons would predict: defensive matches tend to stay defensive on both sides. They built a correction for it, and the correction has been refit on basically every sport that has discrete scoring events. The question is whether rugby looks similar.

When I first ran this heatmap I got a giant (0, 0) cell, three and a half times higher than independence would predict, and I excitedly wrote a paragraph about how rugby's "defensive matches stay defensive" effect is stronger than football's. Then I checked the data. It turned out 133 matches (mostly Top 14 from 2014-15) had the scoring events array empty in the source while still recording scores in the teens or twenties. They were ostensibly "zero tries each, 21-15 final score", which would require seven successful penalty kicks each, which is roughly as common as snow in Marseille. Once I dropped those, the picture is much less dramatic:

{% include figure image_path="/assets/images/champions-cup-final/fig04_dc_signature.png" alt="Tries distribution: observed vs independent" %}

The cleaned (0, 0) cell sits at about 1.0× independence: no lift at all. Most of the grid hovers near 1.0, with a slight diagonal elevation (matches with similar try counts on both sides are mildly more common than independence predicts) and very modestly suppressed asymmetric corners. The football-style Dixon-Coles signature is, in cleaned rugby data, basically absent.

This is already a finding. Whatever model we eventually settle on, we should not feel obliged to start from Dixon-Coles' specific tau correction on the four low-score corners, because the rugby data doesn't have the structure the correction was designed to fix.

## Step two: the model ladder

With no Dixon-Coles signature to chase, the question becomes which rating system best predicts match outcomes. I built a ladder of models from "the simplest thing that could possibly work" up to "I think this might be cute". Each one was scored on the 2024-25 season as a holdout: walk through the season match by match, use only data strictly before each match to predict it, score the prediction. The metric is **Brier score**, which is the mean squared error of probability predictions. A perfect prediction has Brier 0, a coin-flip has Brier 0.25, and the difference between any two models' Brier numbers tells you whether one is meaningfully sharper than the other.

The full ladder:

- **M0 home baseline.** Always predict the empirical home-win rate. The dumbest model that beats a coin flip. Brier 0.200.
- **M1 Bradley-Terry.** A logistic regression with a team dummy for everyone and a home-team dummy. Each team gets one number, the home effect is one number. Brier 0.185. Better than the baseline by 7%, mostly because it knows Leinster is better than Zebre.
- **M2 Glicko-2.** A sequential Bayesian rating per team that updates after each match and carries an explicit uncertainty around each rating. Originally designed by Mark Glickman for chess. Brier 0.172. The cleanest model on the ladder.
- **M3 independent Poisson on tries.** A generative model that fits a try-rate for each team's attack and defence. Predicts the joint try distribution; marginalises to a win probability. Brier 0.192.
- **M4 Dixon-Coles on tries.** M3 plus the corner-cell correction discussed above. Brier 0.192, indistinguishable from M3, because the data didn't have the pattern the correction was designed for.
- **M5 Skellam on try-differential.** Models the try difference directly. Brier 0.219. Worse, because the likelihood throws away information about the joint distribution.
- **M6 negative binomial bivariate.** M3 with a dispersion parameter to absorb overdispersion. Brier 0.192. The dispersion fits to near-zero once team parameters absorb the variance, so it collapses to M3.
- **M8 LightGBM.** A gradient-boosted tree on engineered features: Glicko ratings, rolling form, head-to-head, days since last match. Brier 0.173. Ties Glicko-2 exactly, which is the rare case where adding ten features to a rating differential adds zero signal.
- **M10 Bayesian Dixon-Coles.** M4 with proper priors and NUTS posterior sampling, so we get a credible interval on the win probability. Brier 0.191.

{% include figure image_path="/assets/images/champions-cup-final/fig05_model_ladder.png" alt="Model ladder: Brier on 2024-25 holdout" %}

The headline result: a fifty-line Glicko-2 implementation, tuned for the right home advantage (about 200 rating points, which is much larger than the football literature uses, and which I take as a fact about rugby rather than a quirk of the data), beats every other model in the room. None of the generative bivariate Poisson variants gets close. LightGBM with engineered features matches Glicko-2 to three decimal places. Rating differential is the only feature in our entire engineered feature set that carries signal.

But Glicko alone, fit on twelve years of European club rugby, gave a final-day prediction of about **32% Bordeaux**. The market was at 60%. A near-thirty-percentage-point gap on a market this size is uncomfortable to look at without first stress-testing the model.

## Step three: where might the model be wrong?

If you and the market disagree by thirty points, one of two things is true. Either you have found a real edge, or your model is missing something the market is pricing in. Before I trusted the prior, I wanted to test the second possibility properly. Each hypothesis had the same up-front test: does it improve Brier on the 2024-25 Champions Cup matches specifically by at least 3% relative, with no calibration degradation? A model change that fails the test is documented and rejected, even if it would have moved the prediction in a direction I'd have preferred.

The hypotheses I ran through:

**Squad rotation.** Leinster lost three URC matches early in 2025-26, including a 0-35 hammering by the Stormers in Cape Town. This matters because URC teams (especially the Irish provinces) rotate their squads heavily on the South African tour and during international windows: their first-choice fly-half is in Test camp, their hooker is rested before the Champions Cup, and what you see on the field on a wet Friday in Cape Town is closer to the second-choice XV than the matchday-23 you'll see in Bilbao. Top 14 teams rotate too, but the rotation is more uniform across the season because there's no equivalent international break and the bench rotates more than the starting fifteen. If Leinster's URC losses were B-team losses, the model is over-discounting their first-team strength. I parsed lineups from the source, built an "A-team" for each club from their Champions Cup starts (because no one rotates in the Champions Cup pool stage), and computed each match's overlap with the A-team. The Stormers and Bulls losses were indeed heavily rotated; the Munster loss was full-strength. Building a rotation-weighted Glicko widens the model's gap with the market by a couple of percentage points and doesn't improve the cup Brier. Diagnostically valuable, but not a model improvement.

**Recency decay.** Maybe twelve-year-old matches shouldn't count as much as last weekend's. Glicko-2 already implicitly favours recent matches through its rating-deviation mechanic, but I tried explicit exponential decay with half-lives from two months to three years. Every single value made the cup-holdout Brier worse. The lesson is that old matches are evidently still informative once Glicko's mechanism has processed them; piling on more decay just throws away signal.

**Per-team cup specialism (win/loss based).** Maybe some teams elevate their game in Europe in a way Glicko doesn't capture, and the model should give them a rating bonus for cup matches specifically. I computed each team's residual win rate in cup matches relative to Glicko expectation and used it as an offset. Brier worsened uniformly.

**Style-aware ML.** A LightGBM with engineered "style" features (rolling tries scored and conceded, rolling point differential, head-to-head record). Brier worsened on the cup holdout and the final prediction *dropped* Bordeaux from 39% to 30%. Adding features to predict on a thin slice of the data is a great way to overfit.

Four hypotheses, four rejections. The model and the market were starting to look genuinely at odds rather than tunable.

## Step four: actually testing the right hypothesis

A specific theory then emerged that turned out to be correct: maybe **Bordeaux's domestic record looks ordinary because they play in a harder league, and the model is failing to capture how dominant they have been in Europe.** Two testable claims inside this:

First, Bordeaux's Top 14 opposition is stronger than Leinster's URC opposition. I tested this directly. Bordeaux's average Top 14 opponent in 2025-26 had a Glicko rating of 1628; Leinster's average URC opponent had a rating of 1523. Bordeaux played 105 rating points worth of stronger opposition on average in their domestic league. Glicko-2 already adjusts for opponent rating in the per-match update, so this on its own is not a model flaw. But it is the table-setter for the second claim.

Second, **Bordeaux's margin of victory in the Champions Cup is meaningfully bigger than Leinster's**. Glicko-2 ignores margin entirely: a 64-14 demolition and a 1-point squeak count as the same "win". In a season where one finalist beat Leicester 64-14, Toulouse 30-15, and Bath 38-26, while the other won by 1 against La Rochelle and 4 against Toulon, ignoring margin throws away signal you can see with the naked eye.

A picture is worth a few hundred words. Each dot below is a Champions Cup or Challenge Cup match between 2014 and 2026, plotted as `(rating differential going in)` versus `(actual margin)`. The dashed line is the margin Glicko's rating differential would predict. Points above the line are matches where the team won by more than expected; points below are matches where the team underperformed its rating.

{% include figure image_path="/assets/images/champions-cup-final/fig02_cup_overperformance_scatter.png" alt="Bordeaux beats Champions Cup opponents by more than Glicko expects" %}

The orange dots (Bordeaux) sit reliably above the diagonal: they win their European matches by more than the rating differential predicts. The blue dots (Leinster) are roughly on the line: they win the matches Glicko says they should win, by about the margins Glicko predicts. Bordeaux's overperformance is not specific to 2025-26 either; it has been a multi-year pattern.

The right way to use this is to bake margin of victory into the rating system. I built a Glicko-2 variant with a margin-of-victory adjustment (the standard FiveThirtyEight-style scale: a multiplier of `ln(margin + 1)` modulated by the rating-difference term that prevents stronger teams running up the score for free against weak opposition). The empirical result was unambiguous:

- Brier on 2024-25 Champions Cup matches: 0.147 → 0.140 (5% relative improvement)
- Calibration on the same: ECE 0.10 → 0.07
- Brier on full 2024-25 holdout: 0.172 → 0.170 (no degradation on non-cup matches)

A clean pass. The cup-specific improvement is real, the calibration improves, the rest of the data isn't penalised.

And the bit I trust most about this result: the same model change, applied to the Challenge Cup final the night before (Ulster v Montpellier), brought my model's prediction onto the bookmakers' line almost exactly (model 75%, market 75% for Montpellier). The same change that closes half the Champions Cup gap closes the entire Challenge Cup gap as a side effect. That is the signature of a real signal, not a tune-to-target.

## The journey, in one chart

If you stitch together every change to the model that I tried, the prediction moved like this:

{% include figure image_path="/assets/images/champions-cup-final/fig03_waterfall.png" alt="Model prediction evolution: from 32% to 45%" %}

Each step is a separate, principled change. Some moved the prediction toward the market and some moved it the other way. The whole journey is a 13-percentage-point shift, and it leaves the model 15 percentage points short of the Polymarket midpoint.

Some bonus credibility for the model: looking at every European cup match in the panel, the top cup overperformers by margin are Edinburgh and Benetton. This makes domain sense. Edinburgh have a poor URC record but tend to upset bigger teams in Europe (Edinburgh-style: tight defence, aggressive set-piece, low-margin wins). Benetton are similar: nobody fears Benetton in Italy, but they have made nuisances of themselves to stronger sides in cup matches for years. If the model were just preferring teams I want to bet on, it would be flagging Leinster as an overperformer. It isn't. Bordeaux is fifth on the list at +4 points per match.

{% include figure image_path="/assets/images/champions-cup-final/fig07_cup_dominance.png" alt="European cup overperformance: Bordeaux is top-5" %}

## What the model says, in the end

{% include figure image_path="/assets/images/champions-cup-final/fig01_market_vs_model.png" alt="Market vs model on the 2026 final" %}

After everything: the working model puts Bordeaux at 45%. The market says 60%. The Bayesian Dixon-Coles model (which doesn't have margin of victory yet) is at 33% with a 90% credible interval of [28%, 39%]. The vanilla Glicko-2 (also no margin) is at 39%. The market sits outside the Bayesian DC's 95th percentile, which is unusual on a model I trust.

{% include figure image_path="/assets/images/champions-cup-final/fig08_posterior_density.png" alt="Bayesian DC posterior on P(Bordeaux)" %}

Some thoughts on what might still be moving the remaining gap, as candidates for the next iteration rather than as a list of certainties:

1. **Bilbao is closer to Bordeaux than to Dublin.** The neutral-venue flag in my data is binary. There is a real chance the crowd in Bilbao is worth a few percentage points to Bordeaux, and none of that is in the prior.
2. **The market sees things I don't.** Team news, injuries, the referee assignment, training reports. Polymarket has been trading this match for weeks, and a Tuesday-morning news event is in the price by Tuesday afternoon.
3. **Recent-cup-form effects beyond margin of victory.** Bordeaux qualitatively looks more dominant than my model captures; whether "looked dominant" is more than a margin adjustment is genuinely open.
4. **The market might also be wrong.** Markets are usually informed but not always right. The 15-point disagreement is small enough to be plausibly explained by either side being marginally off.

I'm not going to keep tuning to close the rest. I tested seven candidate model improvements; one held up; the others are documented as having been tested and failed. The remaining gap will get closed during the week by the Bayesian update, not by my keep nudging the prior.

On the sister match: Friday at 21:00 local, Ulster play Montpellier in the Challenge Cup final on the same pitch. Both the bookmakers and my model have Montpellier as a clear favourite at 75%. The agreement is itself useful information: in a match where the analytical answer is straightforward, the model lands where the market lands. The disagreement on the Champions Cup is not a generic "my model is broken" finding.

## What I do

Here is the awkward bit.

I want Bordeaux to win. I will spend Saturday afternoon in Bilbao in a yellow-and-black shirt, with a beer, hoping the defending champions hold their crown. I have a real rooting interest in the model being wrong.

The model says Leinster, mildly. The market says Bordeaux, firmly. I have spent two weeks building the model precisely so that I'd trust it more than my instincts on questions like this. If I were trading this from a cold start I would absolutely fade the market: the model has been tested in every reasonable way I could think of, by metrics chosen in advance, on a holdout chosen in advance, and the disagreement is robust.

So I will probably bet against the team I'm driving four hundred kilometres to support. Small size. Enough to carry some skin, not enough to spoil the beer. The neat thing about this kind of position is that it's a hedge in the emotional sense as well as the analytical one: if Bordeaux win, I'm celebrating in the stands with the team I love and lose a small bet; if Leinster win, I'm sad in the stands but the model and the bet make me whole. Win-win, on a logarithmic utility function.

What I tell anyone who asks about this is the same thing: the model has a job, and your gut has a different job, and you're not supposed to confuse them. If you've built a careful prior and the market is offering you a price outside your 90% credible interval, the framework is telling you something. The honest read of the math is to take the price.

Whether that survives kickoff on Saturday afternoon is a different question.

---

*See you in Bilbao.*
