# I Gave My Data Pipeline a Contract. It Broke on Day Two. That Was Fine.

**Published:** 2026-06-08
**Medium:** https://medium.com/@leelasaikiran4/i-gave-my-data-pipeline-a-contract-it-broke-on-day-two-that-was-fine-4f722d2cac7f
**Read Time:** ~9 min
**Tags:** data-engineering, data-contracts, llm, pydantic, kafka
**Series:** ICU Active Data Systems — Part 3 of 3

---

Schema contracts are a solved problem. You write a Pydantic model, add range checks, wire up a dead-letter topic. Done in an afternoon. The thing that's not solved — and that nobody talks about enough for LLM pipelines specifically — is semantic drift. Your data passes every validation check and still means something different than your model was trained to expect. That failure is invisible until the model starts doing strange things, and by then you've got no obvious place to look.

I added a contract layer to the ICU pipeline from my last two posts to find out how bad the gap actually was. The schema contract broke on day two, which was fine. The semantic gap never triggered a single alert. **That's the problem worth writing about.**

Here's what happened. An upstream field got renamed without notice. The feature engineering stage quietly started producing nulls. Not errors. Nulls. The pipeline stayed green. The Kafka consumer kept running. The LLM downstream got a context window full of missing values and produced confident-sounding nonsense. The contract caught this on day two, exactly as designed. But a subtler failure — same field name, different computation method, valid float — would have passed everything and fed the model data that looked right and wasn't.

## The hole in the previous architecture

My sepsis post ended with two open things. One was prompt versioning, and I built a partial answer with MLflow. The other I kind of glossed over.

Feature engineering was the fix that actually made the sepsis detector work. Pull the temporal math out of the LLM call, pre-compute rolling averages and trend deltas, send the agent something it can reason over. That part was right. What I didn't say: I had zero formal definition of what the feature engineering stage was supposed to *receive*. The upstream vitals stream could change shape at any time and nothing would stop it. I was just hoping it wouldn't.

Data contracts are the answer to that. They're also harder to implement correctly than the blog posts make them sound.

## What I actually built

The definition I keep seeing is clean: a formal agreement between producer and consumer specifying schema, semantics, freshness, and expected volume. The implementation is messier.

Here's the validation layer I put between the raw Kafka topic and the feature engineering worker:

- Pydantic schema validation with domain-specific range checks
- A freshness assertion (message timestamp must be recent)
- Failures go to a dead-letter topic
- The pipeline emits a metric on every validation failure

Three things surprised me running this on MIMIC-IV replay data.

**The contract caught things I didn't know to look for.** I assumed heart rate would always be numeric. It wasn't. Monitor dropouts produced occasional string entries (`"---"`). My feature engineering stage had been silently coercing these to NaN and computing rolling averages on partial windows, with no signal that anything was off.

**The freshness check broke replay immediately.** Historical ICU timestamps are not current timestamps, so `not_stale` failed on everything. I had to add an environment flag — `CONTRACT_MODE=replay` vs `CONTRACT_MODE=live` — that relaxes freshness validation in replay contexts. This felt like cheating at first. It's actually just correct. Production and replay have different requirements and pretending they don't means replay tests stop catching real problems.

**The dead-letter topic became the most useful debugging surface in the stack.** Every validation failure serialized with the original message and failure reason, in one place, queryable. I looked at it more than any other output. It told me more about upstream data quality in a week than three months of standard pipeline metrics had.

## The part Pydantic can't help with

Schema contracts are solvable. The harder problem is **semantic contracts** — promises about what the data *means* rather than what shape it has.

My vitals stream has a `map_bp` field, mean arterial pressure. The schema contract checks that it's a float between 20 and 180. That passes.

The semantic contract would say: `map_bp` should be computed as `(systolic + 2 * diastolic) / 3`, from the same measurement window as the other vitals in this record, not cached from an earlier reading or pulled from a different monitor.

Here's the concrete failure. One subset of patients in the MIMIC-IV data came from a monitor model that reports MAP as a direct pressure transducer reading rather than deriving it from systolic and diastolic. The values look plausible. They're in the valid range. They pass schema validation. But the model I trained expected the derived formula, because that's what the majority of the training data used. The transducer readings run about 3–5 mmHg higher. Not obviously wrong. But consistently wrong in a directional way that matters when you're computing a SOFA cardiovascular score and looking for deterioration trends.

I found this not through the contract but through the distribution drift check. Six weeks in.

I can't enforce the computation method with Pydantic. What I've been doing instead — which is honestly a workaround and not a solution — is tagging the source on every message and adding a validator that rejects records from known non-standard sources until I've verified their computation method.

This is brittle. What I'd want instead: a way to declare computation semantics in the contract schema itself, so any consumer can inspect how a field was computed and reject it if the method doesn't match what they expect. That tooling doesn't exist in a clean form yet.

## Distribution monitoring as a second layer

The second defense layer I added: track the statistical distribution of every feature over a rolling 24-hour window, alert when it drifts beyond a threshold.

This caught something I'd have missed otherwise. Respiratory rate values were systematically lower in one patient subset — not because those patients were healthier, but because that subset came from a different monitor model with a slightly different reporting range. Same field name. Different calibration. Passed schema validation. The distribution check caught it in hours.

## What contracts actually change

The benefit isn't catching individual bad records. It's that **your pipeline's assumptions stop being invisible**.

Assumptions without contracts live in the heads of whoever wrote the feature engineering code. When an upstream system changes and that person has moved teams, nobody knows what broke until a model starts behaving strangely. I've been the person digging backwards through three services to find the cause.

What contracts don't change: they can't tell you if your data is *correct*, only if it's well-formed. If your assumptions are wrong to begin with, a contract will perfectly validate garbage.

## What I still don't have answers for

**Contract governance across teams.** In my side project I own the producer and the consumer. In an actual data mesh, those are different people with different incentives and different release schedules. Who approves a contract change? Who gets paged when it's violated? The tooling for this is early. The organizational process for it barely exists in most places I've seen.

**Contract granularity for LLM pipelines.** For traditional feature engineering, schema and range checks are probably fine. For a pipeline where an LLM reasons over the data, you probably also need semantic assertions: is this field what it claims to be, computed the way the downstream model was trained to expect? I don't have a framework for this. It feels like the right question, and I'd be surprised if anyone has a clean answer yet.

The previous posts were about making pipelines smarter. This one is about a prerequisite I kept skipping: **before an LLM can reason over data, you have to know the data is what you think it is.** Schema contracts get you part of the way. The semantic layer is where the actual problem lives, and I don't think anyone has fully solved it yet.

---

*Previous: [Part 2 — Why My AI Sepsis Detector Failed](./02-ai-sepsis-detector.md)*
