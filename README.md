# Medium Writing

A curated collection of my technical articles published on Medium — covering Data Engineering, AI/ML, streaming systems, and production LLM pipelines.

**Medium Profile:** [@leelasaikiran4](https://medium.com/@leelasaikiran4)

---

## Article Series

### ICU Active Data Systems *(3-part series)*

Building an agentic AI pipeline on top of ICU streaming data — from a basic Spark deterioration detector through a full sepsis classifier to production-grade data contracts.

| # | Title | Published | Medium |
|---|-------|-----------|--------|
| 1 | [Building Active Data Systems: Agentic AI Meets Spark Structured Streaming](./series/icu-active-data-systems/01-building-active-data-systems.md) | 2026-03-06 | [Read →](https://medium.com/@leelasaikiran4/building-active-data-systems-agentic-ai-meets-spark-structured-streaming-921292fcf298) |
| 2 | [Why My AI Sepsis Detector Failed Until I Stopped Feeding It Raw Data](./series/icu-active-data-systems/02-ai-sepsis-detector.md) | 2026-03-17 | [Read →](https://medium.com/@leelasaikiran4/why-my-ai-sepsis-detector-failed-until-i-stopped-feeding-it-raw-data-8f3d6aceaaf2) |
| 3 | [I Gave My Data Pipeline a Contract. It Broke on Day Two. That Was Fine.](./series/icu-active-data-systems/03-data-pipeline-contract.md) | 2026-06-08 | [Read →](https://medium.com/@leelasaikiran4/i-gave-my-data-pipeline-a-contract-it-broke-on-day-two-that-was-fine-4f722d2cac7f) |

---

## Standalone Articles

### Data Engineering

| Title | Published | Medium |
|-------|-----------|--------|
| [Your Data Pipeline Is Working Perfectly. That's the Problem.](./published/data-engineering/2026-06-01-pipeline-working-perfectly.md) | 2026-06-01 | [Read →](https://medium.com/@leelasaikiran4/your-data-pipeline-is-working-perfectly-thats-the-problem-18c0d5860163) |

### AI / ML

| Title | Published | Medium |
|-------|-----------|--------|
| [I Rechunked Once. It Beat Three Embedding Model Swaps.](./published/ai-ml/2026-06-16-rechunked-once.md) | 2026-06-16 | [Read →](https://medium.com/@leelasaikiran4/i-rechunked-once-it-beat-three-embedding-model-swaps-f902f320948d) |
| [My LLM Agent Ran for Six Hours. It Did Nothing Useful. That Was My Fault.](./published/ai-ml/2026-06-20-llm-agent-six-hours.md) | 2026-06-20 | [Read →](https://medium.com/@leelasaikiran4/my-llm-agent-ran-for-six-hours-it-did-nothing-useful-that-was-my-fault-d43cdfbf99f3) |

---

## Writing Stats

| Metric | Count |
|--------|-------|
| Published Articles | 6 |
| Article Series | 1 |
| Topics Covered | Data Engineering, AI/ML, RAG, Agentic AI, Streaming |

---

## Repository Structure

```
medium-writing/
│
├── published/                        # Standalone articles by topic
│   ├── data-engineering/
│   └── ai-ml/
│
├── series/                           # Multi-part series
│   └── icu-active-data-systems/      # 3-part series on agentic ICU pipelines
│
├── drafts/
│   └── in-progress/
│
├── assets/images/                    # Diagrams and screenshots
├── templates/article-template.md     # Starter for new articles
└── .github/workflows/link-checker.yml
```

---

## Start Reading

New here? The series is the best entry point:

**→ [Start with Part 1: Building Active Data Systems](./series/icu-active-data-systems/01-building-active-data-systems.md)**

Or jump straight to a standalone:

- **Opinion piece:** [Your Data Pipeline Is Working Perfectly. That's the Problem.](./published/data-engineering/2026-06-01-pipeline-working-perfectly.md)
- **RAG practitioners:** [I Rechunked Once. It Beat Three Embedding Model Swaps.](./published/ai-ml/2026-06-16-rechunked-once.md)
- **Agent builders:** [My LLM Agent Ran for Six Hours. It Did Nothing Useful.](./published/ai-ml/2026-06-20-llm-agent-six-hours.md)
