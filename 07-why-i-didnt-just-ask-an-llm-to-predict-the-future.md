# Why I Didn't Just Ask an LLM to Predict the Future

A traditional dashboard can only look backward. Once I had one working, the obvious next question was: what happens in the next seven days? And the fastest path to an answer — the one that took no new infrastructure at all — was to just ask an LLM to predict it. I considered that, and then I didn't do it. An LLM is not a reliable numeric forecaster. It was never trained to model an event-count time series, and asking it to guess a number it has no real basis for is a good way to get a confident-sounding answer that happens to be wrong.

So the forecast became its own thing: a dedicated neural network, built specifically to model how event activity rises and falls over time. This post is about the thinking behind it — the intuition it's built on, the features it actually sees, and how it's structured.

## The core intuition: events feed on themselves

The model is built around a Hawkes process — a class of self-exciting point processes, where an event's occurrence temporarily raises the likelihood of related events happening again soon, and that elevated likelihood then decays back to baseline. It's a natural fit for news event streams. A protest erupts, and the following days typically see a spike in related events, which then gradually tapers off. That excite-then-decay shape is the entire story of a news cycle.

The model combines that classic idea with a Transformer: a Transformer encoder learns which days in the historical window matter most for predicting what comes next, through attention, and a Hawkes-style head takes that context and models the excitation from a recent spike along with its exponential decay. The result is deliberately compact — the final checkpoint is only about 1.56MB. The idea in one sentence, borrowed from the code's own comments: a Transformer encoder reads a fixed daily history window, then a Hawkes-style head predicts a baseline intensity plus a decaying excitation term for the days ahead.

## What the model actually sees: 16 numbers per day

The model doesn't look at raw events. It looks at a 16-dimensional feature vector, one per day, built from three groups of signals.

The first six are base features, each log-scaled or normalized to keep long tails from dominating:

```python
log1p(event_count)                 # event volume (log-scaled)
conflict_events / total            # share of conflict events
cooperation_events / total         # share of cooperation events
avg_goldstein / 10                 # conflict/cooperation intensity, normalized
avg_tone / 10                      # media sentiment, normalized
log1p(total_articles)              # coverage volume (log-scaled)
```

The next four are calendar features, encoded with sine and cosine instead of raw integers, specifically to avoid a day-of-week value jumping discontinuously from 6 back to 0:

```python
sin/cos(2π · day_of_week / 7)      # weekly cycle
sin/cos(2π · day_of_year / 366)    # yearly cycle
```

The last six are rolling statistics, meant to capture trend, volatility, and sudden spikes:

```python
log1p(mean_7 / mean_14 / mean_30)  # moving averages at three scales
spike_ratio = current / mean_7     # how far today deviates from the recent baseline
volatility  = std_7 / mean_7       # recent volatility
trend       = slope_7 / mean_7     # 7-day slope
```

The idea running through all sixteen dimensions is the same: convert absolute quantities into relative, log-scaled quantities. A global daily event count and a single actor's daily event count live on wildly different scales, but once both are expressed as log-transformed ratios and rolling comparisons instead of raw counts, they become comparable — and trainable in the same model.

## The structure behind the numbers

On top of the daily feature vector, the model carries a set of embeddings that let one architecture handle multiple prediction targets at once. A series embedding identifies what's actually being predicted — a specific country, actor, or actor pair. An event-type embedding supports multiple tasks: all events, conflict only, cooperation only, or protests specifically. A series-group embedding tracks which scope the prediction belongs to — global, country, actor, country-pair, actor-pair, event-root, or event-code.

The historical window itself defaults to 30 days, though the sweep I'll cover in the next post found that 14 days actually performs better — a detail that turns out to matter more than it sounds like it should once we get to the results. A Transformer encoder with attention pooling learns which of those historical days carry the most weight, and the Hawkes-style head turns that into a baseline intensity plus an excitation term with exponential decay. A multi-step head then outputs all seven days of the forecast horizon directly.

The final checkpoint — versioned `thp_v5_series_event_normalized+calibrated`, trained with CUDA mixed precision — covers 736 distinct target sequences across the four event types.

One design choice matters more than any architectural detail: the model outputs a range, not a single number. Every forecast comes back as a low, median, and high estimate, which is what lets the Forecast page show an actual uncertainty band instead of a single figure dressed up to look more precise than it is. Given how volatile a news event stream actually is, presenting one number as "the" prediction would have been a kind of dishonesty the interval avoids.

---

Designing the model was the easier half of this work. Getting it to actually perform — running a real hyperparameter search, calibrating its output, and being honest about where it does and doesn't work well — is where the harder engineering happened, and where the headline number for this whole project came from: cutting the prediction error in half. That's the next post.
