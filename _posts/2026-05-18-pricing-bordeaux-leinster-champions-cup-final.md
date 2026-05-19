---
layout: single
title: "I built a Bayesian model for the Champions Cup final. It says I should bet against my own team."
date: 2026-05-18
categories: [prediction-markets]
tags: [bayesian, rugby, Champions Cup, Polymarket, Bordeaux, Leinster, glicko]
excerpt: "Polymarket has Bordeaux at 60%. The bookmakers price them as a near-lock. My model, after two weeks of being built and broken and rebuilt, has them at 45%. Here is how the disagreement got that wide, and how it closed."
header:
  image: /assets/images/champions-cup-final/header_ubb_action.jpg
  image_description: "Union Bordeaux Bègles in action against Stade Français (Top 14, January 2025)"
  caption: "Stade Français vs Union Bordeaux Bègles, Top 14, January 2025 (photo: [Like tears in rain](https://commons.wikimedia.org/wiki/File:2024-25_Top_14_Stade_fran%C3%A7ais_vs_Union_Bordeaux_B%C3%A8gles_(617).jpg), [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/))"
  teaser: /assets/images/champions-cup-final/header_ubb_action.jpg
toc: true
toc_sticky: true
mathjax: true
---

I'm driving to Bilbao on Friday to watch Bordeaux Bègles play Leinster in the Champions Cup final. I want Bordeaux to win.

Polymarket has them at 60.5%. William Hill at 77%. Coral at 75%. The market thinks the team I want to win is the team that ought to win.

My model has Bordeaux at 45%. The slight favourite, in my numbers, is Leinster.

Here's how I got there.

{% include figure image_path="/assets/images/champions-cup-final/fig00_hero.png" alt="Probability that Bordeaux wins, by source" %}

## Why the prices are different

The three quoted numbers are 60%, 75%, and 77%, all answering the same yes/no question. They differ for good reasons.

William Hill quotes Bordeaux at 3/10 and Leinster at 12/5. Convert to implied probabilities and they add to **106.3%**. The extra 6.3 points is the **overround**, the bookmaker's structural edge. Bet £100 on each side at fair odds and you would walk out with your £200; at William Hill's odds you walk out with £194. Coral's overround on the same match is 6.0%. Strip the overround out and William Hill still has Bordeaux at 72%.

Polymarket has no house. The market is peer-to-peer, the Yes and No sides add to 100.5%, and the price is whatever the marginal trader is willing to risk capital on. The 60.5% is the consensus of people who have actually committed money, not a defensive line set by a trading desk worried about retail flow. UK bookmakers tend to overprice the favourite on big public events: most casual money backs the favourite, the book shortens the favourite to discourage one-sided exposure, and the price drifts into a territory the bookmaker would not, deep down, defend on its merits.

For the rest of the post, when I say "the market", I mean **Polymarket at 60.5%**. That is the number I would have to take a position against.

## Building a prior

Before I look at the market, I want my own number. The goal is a **prior**: what I would say about `P(Bordeaux wins)` based on the matches I have data for, with no peek at the bookmaker line.

The training data is twelve years of European club rugby across four competitions: Top 14, Premiership, URC, and the Champions Cup itself. After cleaning, the panel is **5,471 matches with home and away scores, tries, cards, and venue**. Each match is one observation: who played whom, where, and what the score was.

I tried nine different rating systems on that data, from a coin flip up through Bayesian Dixon-Coles via NUTS. The way to compare them is **Brier score**: the mean squared error of the predicted probability. A perfect model has Brier 0, a coin flip has 0.25, and the difference between two models is how much sharper one is than the other. I evaluated everything walk-forward on the 2024-25 season as a holdout: each match was predicted using only data strictly before it.

{% include figure image_path="/assets/images/champions-cup-final/fig05_model_ladder.png" alt="Model ladder: Brier on 2024-25 holdout" %}

A 50-line Glicko-2 implementation beats everything fancier in the room. Glicko-2, designed by Mark Glickman for chess, gives every team a rating around 1500 with an uncertainty attached to it. After each match the rating updates by a Bayesian rule that uses the opponent's rating, the result, and the time since the last update. Nothing else. With the home advantage tuned to **200 rating points** on a 2022-24 validation window (much larger than football's, which I take as a real fact about rugby), Glicko-2 lands at Brier 0.172. Bradley-Terry, three Dixon-Coles variants, a Skellam, a negative binomial, and a gradient-boosted tree on engineered features all sit behind it. Rating differential is the only feature I tried that carries any signal at all.

But Glicko, fed twelve years of European rugby, predicts **Bordeaux at about 32%** for Saturday. The market is at 60%. A near-thirty-point gap on a market this size deserves stress-testing before I trust it.

## What I tried that didn't work

Confession: at this point, what I was really doing was trying to bribe the model into preferring Bordeaux. Every candidate change below is in principle a defensible piece of statistical hygiene. In practice I was hoping one of them would make the model agree with my heart. The bar was the same throughout, set in advance: each candidate had to improve Brier on the 2024-25 Champions Cup matches by at least 3% relative, without degrading calibration.

**Squad rotation, Bordeaux side.** Top 14 is a 26-round league plus playoffs against URC's 18 regular-season rounds. Bordeaux plays a third more domestic matches than Leinster in a normal season, more in a year that reaches a Champions Cup final, and any reasonable coach rests key players around the big European weekends. Bordeaux's 14W 10L 1D Top 14 record might look ordinary partly because some of those losses are tactical. I built a rotation-weighted Glicko using lineup data: parse each matchday squad, compare to that club's Champions Cup starting XV (since nobody rotates in the pool stage), and weight matches by how close the on-field side was to the A-team. The data showed Bordeaux rotating modestly around European fixtures, less than Leinster does in URC, but the rotation-weighted model failed the cup-Brier test. The hypothesis is real; the correction doesn't help on the holdout.

**Recency decay.** Maybe old matches should count less. I tried half-lives from two months to three years. Every value made cup-holdout Brier worse. Glicko-2's built-in mechanism already does enough recency adjustment; piling on more decay throws away signal.

**Per-team cup specialism on wins and losses.** Maybe some clubs elevate themselves in Europe and the model should give them a rating bonus for cup matches. I computed each team's residual cup win rate above Glicko expectation, used it as an offset, and tested. Brier worsened across every offset scale.

**Style interactions.** Maybe certain playing styles match up unevenly: a pick-and-go forward pack might struggle against a wide-running attack, or vice versa, in ways that rating differential can't capture. I fed a gradient-boosted tree both teams' rolling tries scored, tries conceded, points conceded, and head-to-head record, so the model could find interactions like "high-attack team vs high-defence team" if they existed. Cup Brier got worse and the prediction on Saturday's final dropped Bordeaux from 39% to 30%. There may be style-matchup effects in rugby, but they're not strong enough in our feature set to beat the rating differential.

Four hypotheses, four rejections. The model and the market were starting to look like an honest disagreement.

## The hypothesis that worked

The candidate that finally moved the prediction came in two parts: **Bordeaux plays in a harder league, and they win their European matches by bigger margins than the model knows about.**

The first part is easy to check. Bordeaux's average Top 14 opponent in 2025-26 had a Glicko rating of **1628**. Leinster's average URC opponent was at **1523**. Bordeaux faced 105 rating points worth of stronger weekly opposition. Glicko-2 already discounts for opponent rating in the per-match update, but it does so under the assumption that a win is a win. Which brings me to the second part.

Glicko-2 ignores margin entirely. A 64-14 demolition and a 1-point heart attack count the same in the rating update. In a season where one finalist beat Leicester 64-14, Toulouse 30-15, and Bath 38-26, while the other crept past La Rochelle by a single point and Toulon by four, ignoring margin throws away something you can see with the naked eye:

{% include figure image_path="/assets/images/champions-cup-final/fig09_path_to_final.png" alt="Both teams' path to the final, by margin of victory" %}

Bordeaux didn't just win their three knockouts. They averaged a 25-point margin doing it, including a quarter-final dismantling of the team currently top of Top 14. Leinster averaged 17 points across theirs, and one of their wins was a one-score game with about a minute left.

The pattern holds across the full panel, not just this season. Take every European cup match in the last twelve years, work out the margin Glicko would have predicted from the rating gap, and subtract it from the actual margin. The result, per team, is a distribution of "points scored above expectation":

{% include figure image_path="/assets/images/champions-cup-final/fig02_cup_overperformance_scatter.png" alt="Bordeaux's CC matches sit clearly above expectation; Leinster's sit on it" %}

Bordeaux's mean sits well to the right of zero. Leinster's mean sits exactly on zero. The two of them have played different Champions Cups for a decade: Bordeaux's wins have been bigger than their rating would suggest, Leinster's precisely as big as it predicts. Edinburgh and Benetton, who play in the URC but make nuisances of themselves in Europe, sit further right than Bordeaux. It is not the pattern you would expect if I were tuning the model to flatter my team.

{% include figure image_path="/assets/images/champions-cup-final/fig07_cup_dominance.png" alt="European cup overperformers, multi-season" %}

The right way to use this is to bake margin of victory into the rating system. I added a FiveThirtyEight-style scaling to the Glicko update: rating changes get multiplied by `ln(margin + 1)` modulated by a term that prevents stronger teams running up the score against minnows for free. Tested on the 2024-25 Champions Cup holdout:

- **Brier: 0.147 → 0.140 (5% relative improvement)**
- **ECE: 0.10 → 0.07 (better calibration)**
- **Full-season Brier: 0.172 → 0.170 (no degradation outside cup matches)**

A clean pass on every metric I had picked in advance. The change held up on a sister validation too: applied to the Challenge Cup final the next night (Ulster vs Montpellier), the same model gives Montpellier 75%, exactly the bookmaker line. The same single change that closed half the Champions Cup gap closed the entire Challenge Cup one as a side effect. That is what a real signal looks like, not a tune-to-target.

## The journey

{% include figure image_path="/assets/images/champions-cup-final/fig03_waterfall.png" alt="How the prediction moved from 32% to 45%" %}

After everything, the working model puts Bordeaux at **45%**. The market is at 60%. The gap has halved.

The remaining 15 points are open. Polymarket has been trading this match for weeks, so it sees team news and training reports my model doesn't. There may be cup-form effects beyond margin of victory that I haven't captured. The market might just be a few points off. The Bayesian framework handles exactly this kind of standing disagreement: over the next five days, as new Polymarket prices print, I update toward the market in proportion to how much it has moved. If the consensus tightens onto my prior, I take the offer. If it doesn't, I sit out.

{% include figure image_path="/assets/images/champions-cup-final/fig08_posterior_density.png" alt="Bayesian DC posterior on P(Bordeaux)" %}

## What I do

The model says Leinster, mildly. The market says Bordeaux, firmly. I have spent two weeks building the model precisely so I would trust it over my instincts.

So I will probably bet against the team I am driving four hundred kilometres to support. Small size. Enough to carry some skin, not enough to spoil the beer. The pleasure of a contrarian bet on your own team is that it works either way. If Bordeaux win, I am in the stadium losing a bet I would happily pay to lose. If Leinster win, I am quietly devastated, but the bet pays for the consolation beers, and with luck the next morning's as well. Win-win, on a sufficiently logarithmic utility function.

The whole point of building one of these is that the model has a job and your gut has a different job, and you do not get to confuse them. If you have built a careful prior and the market is offering you a price outside your 90% credible interval, the framework is telling you something. Take the price.

Whether the math survives kickoff is a different question.

---

*See you in Bilbao.*
