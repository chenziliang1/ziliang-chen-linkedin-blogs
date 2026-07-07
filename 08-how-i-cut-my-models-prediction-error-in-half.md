# How I Cut My Model's Prediction Error in Half

The headline number for this entire project came out of this phase: MAE dropped from 167.74 to 83.77, a reduction of just over 50%. That number didn't come from one clever idea. It came from a training pipeline built to actually test ideas systematically, a hyperparameter sweep that ran eight configurations and let the data pick a winner, and a discipline about what the result actually meant once I had it.

## Building a pipeline that could survive without a checkpoint

Before getting to results, the training setup itself needed to handle a few things that have nothing to do with model architecture but matter just as much. The training script supports GPU training with mixed precision, `torch.compile`, early stopping, dataset caching so the same data doesn't get rebuilt on every run, per-category evaluation, and — most importantly — rolling-origin backtesting, where the model is repeatedly retrained on an expanding historical window and tested on the period right after it. That's a more honest test than a single train/test split, because it mirrors how the model would actually be used: always predicting forward from whatever data exists at that point in time.

The one design decision I'd call out specifically is the fallback logic on the serving side. The service loads a checkpoint from a configured path by default. If that checkpoint exists, it uses the trained neural model. If it doesn't, it falls back to an empirical Transformer-Hawkes forecaster instead of failing outright. That single fallback path means a fresh clone of this repository, with no model weights at all, still returns a forecast instead of a server error. It's a less accurate forecast, but an available one — and an API that degrades gracefully is worth more than one that occasionally doesn't respond at all.

## What eight configurations actually taught me

The sweep tested eight hyperparameter configurations, ranked by a composite score where lower is better:

| trial | score | MAE | RMSE | seq | d_model | layers | lr | dropout |
|---|---|---|---|---|---|---|---|---|
| **5 ★** | **114.01** | **83.77** | 604.71 | 14 | 96 | 3 | 0.0007 | 0.1 |
| 7 | 115.32 | 84.26 | 621.16 | 14 | 64 | 2 | 0.001 | 0.05 |
| 1 | 116.08 | 84.69 | 627.71 | 14 | 64 | 2 | 0.001 | 0.1 |
| 2 | 116.67 | 85.92 | 614.90 | 14 | 96 | 2 | 0.0007 | 0.05 |
| 3 | 120.72 | 88.61 | 642.31 | 21 | 64 | 3 | 0.0007 | 0.05 |
| 6 | 121.57 | 89.35 | 644.40 | 21 | 96 | 2 | 0.0007 | 0.05 |
| 8 | 122.15 | 90.33 | 636.36 | 21 | 96 | 2 | 0.0007 | 0.1 |
| 4 | 122.17 | 89.62 | 650.99 | 30 | 64 | 2 | 0.001 | 0.1 |

Trial 5 won: a 14-day sequence window, model dimension 96, 3 layers, 4 attention heads, a learning rate of 0.0007, batch size 1024, dropout 0.1.

The number that caught my attention wasn't the winning score — it was the `seq` column. Every one of the top four trials used a 14-day history window. The three trials using 21 or 30 days all scored worse, regardless of what else was tuned around them. That's not a coincidence I'd have guessed at ahead of time; it would have been easy to assume more history is always better context for a forecasting model. Here, the opposite held: recent signal carried more predictive weight than a longer lookback, which actually lines up with the Hawkes intuition from the last post — an excitation effect is a short-term phenomenon, so a model built to capture short-term excitation apparently does better when it isn't given a long window diluting that signal.

The other pattern worth naming: trial 7, with a smaller model (64 dimensions, 2 layers) landed within a point of the winner. Going bigger helped, but only marginally — a useful data point for not over-investing in model size once the returns start flattening out.

## A small, nearly free improvement

Every trial also went through residual calibration — a post-hoc correction that adjusts for systematic bias in the raw predictions, computed across 95 distinct target scopes. For the winning configuration, this took the MAE from 84.83 down to 83.77. That's a modest gain on its own, but it came at close to zero additional cost, and it was consistent enough that it earned a permanent spot in the pipeline — the model version's `+calibrated` suffix marks exactly this step.

## What "cut in half" actually means, and where it doesn't apply

On the held-out validation window, the final numbers looked like this:

| Metric | THP neural model | 7-day moving average baseline | Improvement |
|---|---|---|---|
| MAE | 83.77 | 167.74 | ↓50.06% |
| RMSE | 604.71 | 907.72 | ↓33% |
| MAPE | ~35.6% | — | — |

The model also beat a naive last-value baseline and an empirical Hawkes baseline by similar margins, but the comparison against a 7-day moving average is the cleanest way to state the headline: the model's typical error is roughly half of what a simple, reasonable baseline produces.

There's a caveat I think is worth stating as clearly as the headline number itself: this result holds at a global level, and it does not automatically transfer to every narrower prediction target. Global event volume is high and relatively stable, which makes it easier to predict well. A narrow scope — a specific actor pair, or a rare event type like protests — has far fewer samples and much higher variance, and the error on those targets is genuinely larger. Global MAE and narrow-scope MAE aren't measured on the same effective scale, and treating them as comparable would let one good global number quietly paper over instability somewhere much more specific. I'd rather report that honestly than let the 50% figure imply more than it actually covers.

---

That forecast — a median estimate with a low/high range, tagged with whether it came from the real model or the fallback — is what actually reaches the Forecast tab in the frontend. But a number on a page, even an honest one, still needs to be explained to someone asking a plain-language question about it. That's the job of the next layer in this platform, and it comes with its own hard constraint: whatever explains this data isn't allowed to make anything up. The next post is about how that layer is built.
