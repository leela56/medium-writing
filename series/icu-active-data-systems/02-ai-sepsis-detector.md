# Why My AI Sepsis Detector Failed Until I Stopped Feeding It Raw Data

**Published:** 2026-03-17
**Medium:** https://medium.com/@leelasaikiran4/why-my-ai-sepsis-detector-failed-until-i-stopped-feeding-it-raw-data-8f3d6aceaaf2
**Read Time:** ~10 min
**Tags:** ai-ml, data-engineering, llm, clinical-ai, feature-engineering
**Series:** ICU Active Data Systems — Part 2 of 3

---

My last post ended with a working ICU deterioration detector and two honest admissions.

The first: I had teased a confidence-based routing pattern (HIGH goes to a nurse immediately, borderline goes to a review queue) but hadn't actually built it.

The second: I had no clean story for what happens when a prompt update breaks something at 3am. No versioning, no rollback, no audit trail.

Both felt like loose ends I could live with for a side project. Then I tried to extend it to detect sepsis.

It failed.

Not because the prompt was bad. Not because the model was weak.

It failed because I was sending raw ICU data directly to the agent.

**The biggest improvement didn't come from prompt tuning. It came from pre-computing the features before the LLM ever saw the data.**

## Sepsis

*Sepsis is what happens when the body's response to an infection turns against itself. It's not the infection that kills you, it's your own immune system overreacting systemwide. What makes it hard to catch: it doesn't announce itself. It shows up as a lactate creeping up, a blood pressure drifting down, a heart rate that's been quietly climbing for two hours. By the time any single number crosses a hard threshold, you're already behind.*

**This isn't the same architecture with different data.**

The deterioration detector was, at its core, a single-snapshot problem. Look at the current vitals. Look at the current nursing note. Ask the agent if something looks wrong.

Sepsis doesn't work that way.

The clinical definition, Sepsis-3, requires a suspected infection plus acute organ dysfunction. You measure that dysfunction through SOFA score: six organ systems, scored individually, and you're looking for an increase of 2 or more from baseline. A single reading tells you almost nothing. You have to watch the trajectory over hours.

A lactate of 2.1 mmol/L is borderline. Unremarkable, actually. A lactate that was 1.4 three hours ago and is now 2.1 and still climbing — that's the signal Sepsis-3 literature specifically flags as more significant than any absolute value. **The trend is the diagnosis, not the number.**

Three things had to change because of this.

### Feature engineering needed its own stage

In my previous project, raw vitals went straight to the agent. That worked because sudden deterioration is visible in raw values. Sepsis is slower and noisier. The first time I sent raw sensor readings to Claude and asked for a sepsis assessment, the responses were all over the place: reasoning anchored to a transient heart rate spike, inconsistent confidence scores, hallucinated trends.

I had to pull feature engineering out into a dedicated worker that sits between the raw Kafka stream and the agent. Nothing reaches Claude until it's been cleaned, smoothed, and converted into trend deltas. **The agent never sees a raw value. It sees what happened over the last hour or three hours.** That change alone improved response quality more than any prompt tuning I did.

### The join got significantly messier

My previous project joined one vitals stream with one notes stream. Here I'm joining five lab signals with four vitals signals and computing SOFA deltas across a six-hour lookback window. Vitals arrive every twenty minutes. Labs arrive every few hours. Get the watermarking wrong and the agent is evaluating current vitals against a lactate reading from four hours ago. That produces confident-sounding nonsense.

### The routing error costs became asymmetric

In fraud detection a false negative is expensive. In sepsis a false negative is a different category of problem entirely. That asymmetry is why I couldn't leave confidence routing as a theoretical improvement this time.

## The Pipeline: Two workers, one agent, no Spark

The stack is deliberately simpler than my previous project. No Databricks, no Spark. Two Python workers, Kafka as the backbone, Claude 3.5 Sonnet for the reasoning, Supabase as the state store.

```
MIMIC-IV Simulator
 │ raw vitals + labs
 ▼
Confluent Kafka [raw-vitals-topic]
 │
 ▼
Feature Engineering Worker (03_feature_engineering.py)
 │ rolling averages · lab trends · SOFA delta
 ▼
Confluent Kafka [enriched-topic]
 │
 ▼
LLM Agent Worker (04_llm_agent.py)
 │ Claude 3.5 Sonnet → structured JSON → route
 ▼
Supabase (PostgreSQL) → FastAPI → Next.js Dashboard
```

The reason I dropped Spark: the first post proved the pattern works. This one needed to show it doesn't require enterprise infrastructure.

### Stage 1: Feature Engineering — the stage the first project didn't have

Raw ICU data is genuinely noisy. A patient coughing spikes heart rate. A positional blood pressure reading transiently tanks MAP. These aren't clinical events. But a naive pipeline treats them exactly like clinical events, and an LLM will happily reason about them as if they matter.

The feature engineering worker does two things before anything reaches the agent:
- 1-hour rolling averages for vitals. Smooths transient spikes while keeping genuine trends intact.
- 3-hour moving deltas for labs. Converts an absolute value into a direction: rising, stable, or falling.

### The three-tier routing

The thresholds are asymmetric on purpose. Sepsis has a narrow intervention window and the cost of a missed HIGH is categorically different from the cost of an unnecessary alert. I set the HIGH trigger at 0.75 rather than 0.80.

The WATCH tier turned out to be harder than HIGH or LOW. HIGH fires and someone acts. LOW logs and nobody worries. WATCH requires the dashboard to actually be checked, on a cadence, by someone with a protocol. In my simulation I had WATCH cases escalate to HIGH within forty minutes when nobody cleared them. The technology is fine. **The workflow question — who checks the WATCH queue, and how often — is what determines whether this is useful or just another thing that gets ignored.**

## Prompt versioning: from footnote to actual implementation

Between v1 and v2 of the sepsis assessment prompt, I changed the SOFA weighting. v2 weighted cardiovascular and renal failure more heavily, which the Sepsis-3 literature supports. v2 produced better assessments on my test cases.

It also produced three times as many HIGH alerts on a subset of patients who were actually stable. I ran four hours of simulation before I caught it.

Without versioning I'd have had no way to answer: which prompt was running during which window? Which patients got evaluated under v1 and which under v2?

What I implemented:

```python
# every prompt change goes through this before deployment
with mlflow.start_run(run_name=f'sepsis_prompt_v{version}'):
    mlflow.log_param('prompt_version', version)
    mlflow.log_param('model', 'claude-3-5-sonnet-20241022')
    mlflow.log_artifact(f'prompts/sepsis_v{version}.txt')
    mlflow.log_metric('high_alert_rate', high_rate)
    mlflow.log_metric('precision_on_test_cases', precision)
    mlflow.log_metric('sofa_criterion_alignment', alignment)
```

The pipeline reads the active version from a config file at startup. Rollback is a one-line config change and a process restart.

I want to be honest about what this doesn't solve. This is version control for prompts, not change management. In a real hospital, changing a prompt that affects whether a patient gets escalated would need clinical sign-off, documented testing, a review process. None of that tooling exists in a clean form yet. What I built is the minimum foundation: the evidence trail.

## What I Didn't Expect

**Pre-computing the features mattered more than prompt tuning.** I spent probably three hours iterating on prompt language trying to get consistent confidence scores. Then I moved to pre-computed rolling features and the quality jumped immediately. The agent is much better at reasoning over "lactate delta is +0.7 over three hours" than it is at computing that trend itself from raw timestamped readings. **Do the temporal math before handing it the context.**

**Supabase turned out to be a better bridge than Delta Lake.** Compute layer and serving layer are independent. If the pipeline backs up, the dashboard still shows the last known state. I'd make this choice intentionally on the next project, not just because of a platform constraint.

**The data source was less of a bottleneck than I expected.** MIMIC-IV Demo (100 de-identified ICU patients, publicly available) was enough to validate the full pipeline end to end. The watermarking problems, the state management edge cases, the prompt calibration issues — all of them showed up on 100 patients.

## What This Post Still Didn't Solve

**LLM variance at the routing threshold.** Run the same enriched features through Claude twice with temperature = 0 and you might get 0.78 and then 0.71. With a HIGH threshold at 0.75, that's the difference between an immediate alert and a queue flag. The right answer is probably ensemble scoring: three independent calls, majority vote. I haven't built it.

**No ground truth numbers.** The MIMIC-IV Demo doesn't include the sepsis3 derived table from the full release. Right now I can say the HIGH alerts look clinically reasonable when I review them manually. I can't say "this outperforms a pure SOFA threshold system by X%."

**Circuit breaker still isn't built properly.** Right now the agent worker backs off with exponential retry when the Anthropic API goes down. That's not the same as graceful degradation: falling back to pure threshold alerts, logging the gap, resuming cleanly from the last stable state.

## Closing

Confidence routing: built, with asymmetric thresholds calibrated to the error cost. Prompt versioning: built, with MLflow as the evidence trail.

Neither is production-ready without ground truth validation and proper circuit-breaker logic. But both are real implementations now, not footnotes.

The hardest part of prompt versioning isn't the logging. It's that you're treating a piece of natural language as a deployable artifact that affects clinical decisions. **The governance process for that doesn't really exist yet.** The logging is the easy part. The "who approves this change and how do you know it's safe" part is the open problem.

Live demo: https://sepsis-risk-dashboard.vercel.app/
Code: https://github.com/leela56/sepsis-risk-dashboard

---

*Previous: [Part 1 — Building Active Data Systems](./01-building-active-data-systems.md)*
*Next: [Part 3 — I Gave My Data Pipeline a Contract](./03-data-pipeline-contract.md)*
