---
layout: single
title: "I built a Bayesian model for the Champions Cup final. It says I should bet against my own team."
date: 2026-05-18
categories: [prediction-markets]
tags: [bayesian, rugby, Champions Cup, Polymarket, Bordeaux, Leinster, glicko]
excerpt: "Polymarket has Bordeaux at 60%. The bookmakers price them as a near-lock. My model — built over two weeks, broken several times — has them at 45%. Here's how the disagreement got that wide, and how it closed."
header:
  image: /assets/images/champions-cup-final/header_ubb_action.jpg
  image_description: "Union Bordeaux Bègles in action against Stade Français (Top 14, January 2025)"
  caption: "Stade Français vs Union Bordeaux Bègles, Top 14, January 2025 (photo: [Like tears in rain](https://commons.wikimedia.org/wiki/File:2024-25_Top_14_Stade_fran%C3%A7ais_vs_Union_Bordeaux_B%C3%A8gles_(617).jpg), [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/))"
  teaser: /assets/images/champions-cup-final/header_ubb_action.jpg
toc: true
toc_sticky: true
mathjax: true
---

I'm driving to Bilbao on Friday to watch Bordeaux Bègles play Leinster in the Champions Cup final. Like everyone else who has bought a ticket, I would prefer Bordeaux to win.

Polymarket has them at 60.5%. William Hill at the equivalent of 77%. Coral at 75%. The market thinks the team I want to win is the team that ought to win.

After two weeks of building a model, testing it, breaking it, and putting it back together, my model thinks Bordeaux is a 45% shot. The slight favourite, in my numbers, is Leinster.

What follows is the story of how I got there.

{% include figure image_path="/assets/images/champions-cup-final/fig00_hero.png" alt="Probability that Bordeaux wins, by source" %}

## Why the prices are different

The three quoted numbers are 60%, 75%, and 77%, all answering the same yes/no question. They differ for good reasons.

William Hill quotes Bordeaux at 3/10 and Leinster at 12/5. Convert to implied probabilities and they add to **106.3%**. That extra 6.3 percentage points is the **overround**, the structural edge the bookmaker bakes into every market. If you bet £100 on each side at fair odds you'd be guaranteed your stake back; at William Hill's odds you'd be down about £6. Coral's overround on the same match is 6.0%. Strip the overround out by rescaling and William Hill still has Bordeaux at 72% to win.

Polymarket has no house. The market is peer-to-peer, the two sides of the Yes/No pair add to 100.5%, and the price is whatever the marginal trader is willing to risk capital on. The 60.5% is the consensus of people who have actually committed money rather than a price the bookmaker has set with retail exposure in mind. UK bookmakers tend to overprice the favourite on big public events because most casual money backs the favourite, and the book shortens the favourite to discourage one-sided exposure. The Polymarket midpoint is unencumbered by that.

For the rest of the post, when I say "the market", I mean **Polymarket at 60.5%**. That is the number I would actually have to take a position against.

## Building a prior

Before I look at the market, I want my own number. The goal is a **prior**: what I would say about `P(Bordeaux wins)` based on the matches I have data for, with no peek at the bookmaker line.

The training data is twelve years of European club rugby across four competitions: Top 14, Premiership, URC, and the Champions Cup itself. After cleaning, the panel is **5,471 matches with home and away scores, tries, cards, and venue**. Each match is one observation: who played whom, where, and what the score was.

I tried nine different rating systems on that data, from a coin flip up through Bayesian Dixon-Coles via NUTS. The way to compare them is **Brier score**: the mean squared error of the predicted probability. A perfect model has Brier 0, a coin flip has 0.25, and the difference between two models is how much sharper one is than the other. I evaluated everything walk-forward on the 2024-25 season as a holdout: each match was predicted using only data strictly before it.

{% include figure image_path="/assets/images/champions-cup-final/fig05_model_ladder.png" alt="Model ladder: Brier on 2024-25 holdout" %}

The headline finding is that a 50-line Glicko-2 implementation beats everything fancier in the room. Glicko-2 is a sequential rating system designed by Mark Glickman for chess: every team carries a number around 1500 plus an explicit uncertainty around it, and the number is updated after each match by a Bayesian rule that uses only the opponent's rating, the result, and the time since the last update. With the home advantage tuned to **200 rating points** on a 2022-24 validation window (much larger than the football literature would suggest, which I take as a real fact about rugby), Glicko-2 lands at Brier 0.172. Bradley-Terry, three Dixon-Coles variants, a Skellam, a negative binomial, and a gradient-boosted tree on engineered features all sit behind it. The rating differential is the only feature I tried that carries signal.

But Glicko, fed twelve years of European rugby, predicts **Bordeaux at about 32%** for Saturday. The market is at 60%. A near-thirty-point gap on a market this size is not something to publish without first asking the model some hard questions.

## What I tried that didn't work

If my model and the market disagree by thirty points, one of two things is true: I have an edge, or I am missing something. Before trusting the prior, I tested every hypothesis I could think of for what the model might be missing. The bar was the same throughout: each candidate had to improve Brier on the 2024-25 Champions Cup matches specifically, by at least 3% relative, without degrading calibration.

**Squad rotation, Bordeaux side.** Top 14 is a 26-round league plus playoffs, against URC's 18-round regular season. Bordeaux plays roughly a third more domestic matches than Leinster, and in a year where they reached the Champions Cup final they almost certainly rested key players around big European fixtures. Their 14W 10L 1D Top 14 record looks ordinary partly because some of those losses are tactical: B-team selections in the weeks bracketing a European pool match. I built a rotation-weighted Glicko using lineup data (parse the matchday-23, compare to each club's Champions Cup starts, weight matches by how close the on-field XV is to the A-team). Bordeaux's losses around European weekends *do* show meaningful rotation. But the rotation-weighted model fails the cup-Brier test: it doesn't predict cup outcomes better than vanilla Glicko.

**Recency decay.** Maybe old matches should count less. I tried half-lives from two months to three years. Every value made cup-holdout Brier worse. Glicko-2's built-in mechanism already does enough recency adjustment; piling on more decay throws away signal.

**Per-team cup specialism on wins and losses.** Maybe some clubs elevate themselves in Europe and the model should give them a rating bonus for cup matches. I computed each team's residual cup win rate above Glicko expectation, used it as an offset, and tested. Brier worsened across every offset scale.

**Style interactions.** Maybe certain playing styles match up unevenly: a pick-and-go forward pack might struggle against a wide-running attack, or vice versa, in ways that rating differential can't capture. I fed a gradient-boosted tree both teams' rolling tries scored, tries conceded, points conceded, and head-to-head record, so the model could find interactions like "high-attack team vs high-defence team" if they existed. Cup Brier got worse and the prediction on Saturday's final dropped Bordeaux from 39% to 30%. There may be style-matchup effects in rugby, but they're not strong enough in our feature set to beat the rating differential.

Four hypotheses tested, four rejections. The model and the market were starting to look genuinely at odds, not tunable.

## The hypothesis that worked

The candidate that finally moved the prediction came in two parts: **Bordeaux plays in a harder league, and they win their European matches by bigger margins than the model knows about.**

The first part is easy to check. Bordeaux's average Top 14 opponent in 2025-26 had a Glicko rating of **1628**. Leinster's average URC opponent was at **1523**. Bordeaux faced 105 rating points worth of stronger weekly opposition. Glicko-2 already discounts for opponent rating in the per-match update, but it does so under the assumption that a win is a win. Which brings me to the second part.

Glicko-2 ignores margin entirely. A 64-14 demolition and a 1-point heart attack count the same in the rating update. In a season where one finalist beat Leicester 64-14, Toulouse 30-15, and Bath 38-26, while the other crept past La Rochelle by a single point and Toulon by four, ignoring margin throws away something you can see with the naked eye:

{% include figure image_path="/assets/images/champions-cup-final/fig09_path_to_final.png" alt="Both teams' path to the final, by margin of victory" %}

Bordeaux didn't just win their three knockouts. They averaged a 25-point margin doing it, including a quarter-final dismantling of the team currently top of Top 14. Leinster averaged 17 points across theirs, and one of their wins was a one-score game with about a minute left.

The same pattern shows up across the panel, not just this season. Every European cup match in twelve years, plotted as `(rating differential going in)` versus `(actual margin)`:

{% include figure image_path="/assets/images/champions-cup-final/fig02_cup_overperformance_scatter.png" alt="Bordeaux's CC matches sit above the line" %}

Bordeaux's orange dots tilt above the diagonal: about 60% of their European matches land on the "won by more than rating predicts" side of the line, with an average overperformance of around five points. Leinster's blue dots cluster on the line — they win roughly the matches Glicko expects them to win, by roughly the margins Glicko predicts. Edinburgh and Benetton, both URC sides who tend to be unremarkable in the league but punch up in cups, are the largest overperformers of all. That last detail does matter: if my model were just confusing "I want this team to win" for signal, it wouldn't be flagging Edinburgh as the league's best cup overperformer.

{% include figure image_path="/assets/images/champions-cup-final/fig07_cup_dominance.png" alt="European cup overperformers, multi-season" %}

The right way to use this is to bake margin of victory into the rating system. I added a FiveThirtyEight-style scaling to the Glicko update: rating changes get multiplied by `ln(margin + 1)` modulated by a term that prevents stronger teams running up the score against minnows for free. Tested on the 2024-25 Champions Cup holdout:

- **Brier: 0.147 → 0.140 (5% relative improvement)**
- **ECE: 0.10 → 0.07 (better calibration)**
- **Full-season Brier: 0.172 → 0.170 (no degradation outside cup matches)**

A clean pass on every metric I had picked in advance. The change held up on a sister validation too: applied to the Challenge Cup final the next night (Ulster vs Montpellier), the same model gives Montpellier 75% — exactly the bookmaker line. The same single change that closed half the Champions Cup gap closed the entire Challenge Cup one as a side effect. That is what a real signal looks like.

## The journey

{% include figure image_path="/assets/images/champions-cup-final/fig03_waterfall.png" alt="How the prediction moved from 32% to 45%" %}

After everything, the working model puts Bordeaux at **45%**. The market is at 60%. The gap has halved.

The remaining 15 points are honestly open. Polymarket has been trading this match for weeks, so it sees team news and training reports my model doesn't. There may be cup-form effects I haven't captured beyond margin of victory. Or the market might just be a few points off. The Bayesian framework is built for exactly this kind of standing disagreement: over the next five days, as new Polymarket prices print, I update toward the market in proportion to how much it has moved. If the consensus tightens onto my prior, I take the offer. If it doesn't, I sit out.

{% include figure image_path="/assets/images/champions-cup-final/fig08_posterior_density.png" alt="Bayesian DC posterior on P(Bordeaux)" %}

## What I do

The model says Leinster, mildly. The market says Bordeaux, firmly. I have spent two weeks building the model precisely so that I'd trust it more than my instincts.

So I will probably bet against the team I am driving four hundred kilometres to support. Small size — enough to carry some skin, not enough to spoil the beer. The neat thing about a small contrarian bet on your own team is that it works either way: if Bordeaux win on Saturday, I am in the stadium losing a bet I'd happily pay to lose; if Leinster win, I am quietly devastated but the model and the bet make me whole. Win-win, on a logarithmic utility function.

The whole point of building this kind of thing is that the model has a job and your gut has a different job, and you are not supposed to confuse them. If you have built a careful prior and the market is offering you a price outside your 90% credible interval, the framework is telling you something. Take the price.

Whether the math survives kickoff is, of course, a different question.

---

*See you in Bilbao.*
