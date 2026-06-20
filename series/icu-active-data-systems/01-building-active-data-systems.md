# Building Active Data Systems: Agentic AI Meets Spark Structured Streaming

**Published:** 2026-03-06
**Medium:** https://medium.com/@leelasaikiran4/building-active-data-systems-agentic-ai-meets-spark-structured-streaming-921292fcf298
**Read Time:** ~8 min
**Tags:** data-engineering, apache-spark, ai-agents, streaming, databricks
**Series:** ICU Active Data Systems — Part 1 of 3

---

I've been a data engineer long enough to have strong opinions about checkpointing strategies and mild trauma around Flink cluster upgrades. Our pipelines have always been good at one thing which is moving data reliably from A to B. What they've never been good at is understanding what they're moving.

Talking to nurse friends over the years, the same complaint comes up repeatedly… and also noted in a 2019 JAMA study which found ICU nurses receive upward of 350 monitor alerts per shift, the vast majority false positives. That number stuck with me and it became personal when I heard a friend who works ICU nights describe the same recurring problem.

A patient's vitals would trend badly for 40 minutes before anyone caught it. Not because anyone was negligent — she was covering 8 beds on a short-staffed overnight shift. The monitor dashboard refreshed every 5 minutes. The math just didn't work in the patient's favor.

I'm a data engineer. Watching streams of data continuously is literally what I build systems to do. So I started a side project to see if I could build something that watched the vitals stream and — more importantly — understood context, not just numbers.

## The Problem With Hard Thresholds

Every ICU monitor already fires alerts when SpO2 drops below 90% or heart rate crosses 120 bpm. That's not the gap I was trying to fill.

The gap is that a patient's SpO2 sitting at 93% means completely different things depending on context. On a stable post-surgical patient, it's worth a note. On a patient whose nurse charted "increased lethargy, family reports confusion started this morning, second episode of diaphoresis this shift" — that's a deterioration in progress. The number is the same. The situation is not.

Nursing notes carry that context. They're written in clipped, abbreviation-heavy clinical shorthand — `"Pt c/o SOB, discussed w/ resident, cont to monitor"` — but they contain exactly the kind of qualitative signal that separates noise from a real early warning. No threshold catches that. A regex doesn't touch it. But an LLM reads it fine.

So the project became: ingest a continuous stream of vitals, join it with nursing notes, and have an AI agent evaluate the combined picture — not just the numbers.

I used the [PhysioNet MIMIC-IV dataset](https://physionet.org/content/mimiciv/2.2/), a de-identified ICU dataset from Beth Israel Deaconess Medical Center, and simulated a Kafka stream by replaying events at 10x speed. Not production — but real enough data to validate whether the architecture actually worked.

## Why This Needed a New Architecture

My first attempt was the obvious one: run a Spark batch job every 5 minutes, score recent vitals windows, write flags to a table. It worked. It was also useless — same latency problem the dashboard already had.

Moving to streaming meant two new headaches.

The first was **logic drift**. To get sub-second latency, the conventional wisdom is to move inference to Flink. But now you have Spark for training and feature engineering, and Flink for serving — two codebases, two versions of your feature logic, slowly diverging. I've seen this kill models in production quietly: the training pipeline computes a rolling 10-minute heart rate average one way, the serving pipeline computes it slightly differently, and your model is scoring on features it was never trained on. On a fraud detection system that's an annoying bug. In a clinical context it's worse.

The second was **getting the output somewhere actionable**. Writing a flag to a Delta table is only useful if something reads it and does something. I needed a closed loop — flag fires, nurse gets paged, not flag fires and sits in a warehouse until someone runs a query.

Both problems have cleaner solutions now than they did two years ago.

## The Stack: Spark Real-Time Mode + Agentic AI + Reverse ETL

**Spark Real-Time Mode (RTM)** gets you to sub-second micro-batch latency without a second engine. One codebase, one API, same feature logic in training and serving. Logic drift goes away because there's nothing to drift against.

**`ai_query()`** is Databricks' native function for calling an LLM inline inside a Spark SQL expression. Instead of shipping data out to a separate inference service, the reasoning happens inside the pipeline itself.

**Reverse ETL** closes the loop. The pipeline writes actionable records to a Delta table; a downstream tool reads that table and fires the actual alert — in this case, a push notification to the charge nurse's station.

One important lesson before the code: when I first called `ai_query()` row-by-row, latency jumped to ~200ms per event and throughput collapsed. The fix was batching LLM calls inside `mapInPandas` — brought it to ~45ms per event.

## The Pipeline

**What it does:**

1. Ingests streaming vitals events from Kafka (heart rate, SpO2, respiratory rate, BP)
2. Joins with the latest nursing note for each patient
3. Passes the combined context to an LLM for deterioration classification
4. Filters on high-risk assessments
5. Writes to Delta — Reverse ETL fires the alert from there

Code: https://github.com/leela56/icu_deterioration_detection_agentic_streaming

One thing I'd add in production: a confidence score in the LLM response, not just a binary label. Prompt it to return JSON like `{"label": "HIGH_RISK", "confidence": 0.92, "reason": "..."}` and you can route borderline cases to a human review queue instead of auto-alerting. Saved us from a nasty false-positive incident with a high-value customer.

## Where This Actually Helps (and Where It Doesn't)

This pattern works well for three types of problems:

**Fraud and risk intervention** — where the cost of a false negative far outweighs the latency budget of a human review loop.

**Claims and support triage** — incoming documents with unstructured content (PDFs, emails, photos) that need to be parsed and routed before they hit a human queue.

**Session-aware personalization** — refreshing recommendations mid-session based on live click behavior, not yesterday's model scores.

Where it *doesn't* help: anywhere your LLM prompt is vague, your output schema is inconsistent, or you haven't stress-tested what happens when the model endpoint goes down. A streaming pipeline with an external LLM dependency needs proper circuit-breaker logic. I learned this the hard way when an API timeout caused a checkpoint stall that took 20 minutes to recover from.

## What This Actually Eliminates

The real win isn't the fancy AI part — it's that you finally have one engine for both batch training and real-time serving. Same Spark API, same feature logic, no Flink cluster to babysit. Logic drift goes away because there's only one codebase.

And by pairing it with Reverse ETL, the warehouse stops being a place data goes to sit. It becomes the thing that triggers actions. That's a meaningful architectural shift, not just a performance upgrade.

## What I Still Haven't Solved

Model versioning in a hot path is genuinely unsolved for me. When I update the prompt or swap the underlying model, I have no clean rollback story if the new version starts misbehaving in production. Blue-green deployment for a streaming LLM call inside Spark is not something I've seen a clean pattern for yet.

If you've built something like this and figured out the model governance piece, I'd actually like to know how you approached it.

---

*Next in series: [Part 2 — Why My AI Sepsis Detector Failed Until I Stopped Feeding It Raw Data](./02-ai-sepsis-detector.md)*
