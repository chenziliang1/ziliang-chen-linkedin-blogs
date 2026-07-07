# What I'd Do Differently: Results, Limits, and Lessons from Shipping This Platform

The first post in this series opened with two numbers: roughly 17 million events, and a prediction model whose error I'd cut in half. At the time, those numbers were a hook — a way to signal scale and payoff before asking for ten more posts of someone's attention. Ten posts later, I can point to exactly where each one came from, and what it actually took to get there. This last post is about closing that loop honestly: what shipped, what its real limits are, and what the whole project actually taught me.

## What actually shipped

The platform runs end to end: Dashboard, Forecast, and Analyst Chat, all working together, started with a single Docker Compose command. It processes roughly 17 million events and 82 million articles of 2024 North American GDELT data. The prediction model's error dropped from an MAE of 167.74 down to 83.77 in rolling backtests — a reduction just over 50%. The database optimization work took the platform's worst query from 45 seconds down to about 0.3 seconds, and brought a full year's time-range query down from 60 seconds to under half a second. Chat answers come back with real data behind them, combining SQL, semantic search, and a visible trace back to the source. The whole thing is packaged into a private repository complete with the final model checkpoint, training logs, and sweep results.

That's the honest version of the two numbers I opened with. They weren't decoration — they were the compressed version of everything the last ten posts actually walked through.

## Where it still falls short

None of that is worth much without an honest account of where the platform's edges actually are.

**Prediction quality depends on having a trained checkpoint and enough history behind it.** A fresh clone of this repository, without the model artifacts or a retraining run, falls back to an empirical Hawkes forecaster — a reasonable approximation, but a meaningfully weaker one than the trained model.

**Narrow-scope predictions are less reliable than the headline number suggests.** Global and country-level forecasts are the easy case — high volume, more stable patterns. A specific actor pair, or a rare event type like a protest sequence, has far fewer samples and much more variance, and the error there is genuinely larger. I flagged this back in the model results post, and it's worth repeating here as an actual limitation, not just a caveat: global MAE and narrow-scope MAE are not interchangeable numbers.

**The semantic search layer needs to be built before it's useful.** If the vector index hasn't been generated, ChromaDB-backed search simply returns nothing — there's no fallback error message pointing at the missing step.

**Complex narrative generation depends on an external LLM API**, which means latency and ongoing cost that aren't fully within the platform's control.

**The BigQuery-backed GKG enhancement is genuinely optional**, and requires its own GCP configuration and cost management to use at all.

None of these are fatal. They're the kind of limitations that come from finishing a real project on a real timeline rather than an idealized one, and I'd rather list them plainly than let the headline numbers imply a system with no edges.

## Where I'd take it next

A few directions stood out as the clear next steps, if this project continued: automating the management of model artifacts instead of handling them manually, scheduling incremental updates for the precomputed tables and the vector index instead of full rebuilds, tightening the uncertainty calibration on the prediction model further, and making actor name normalization more robust across the messier corners of the raw data. None of these are architectural rewrites — they're the natural next layer of polish on a system that already works.

## The one decision that mattered most

If I had to reduce this entire project to a single lesson, it wouldn't be about databases, or Transformers, or Hawkes processes. It would be the decision I wrote about in the second post of this series: walking away from the MCP-centered architecture I'd committed to at the start, once it became clear that the glue holding it together was generating more problems than the product itself.

That decision shaped almost everything that came after it. The fast/slow separation that made the platform responsive, the single SQL source of truth that let a fix in one place benefit the entire system, the four performance patterns that outlived the architecture change entirely, the discipline of validating data contracts at every boundary instead of assuming they'd hold — none of that came from the original plan. It came from being willing to revisit the plan once the evidence said it wasn't working.

That's the part I'd underline for anyone reading this who's mid-project and already suspecting their original design might be wrong: a more sophisticated protocol or architecture isn't automatically the better engineering choice for the problem actually in front of you. Recognizing that, and changing course before the sunk cost gets any larger, isn't a failure to plan well the first time. It's the actual job.

That's the platform, start to finish — what it does, what it doesn't do yet, and the one decision that made the difference between a project that worked and one that didn't. Thanks for reading the whole series.
