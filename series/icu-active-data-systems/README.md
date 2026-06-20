# Series: ICU Active Data Systems

A 3-part series building an agentic AI pipeline on top of ICU streaming data — from a basic Spark deterioration detector through a full sepsis classifier to production-grade data contracts.

| Part | Title | Published | Medium |
|------|-------|-----------|--------|
| 1 | [Building Active Data Systems: Agentic AI Meets Spark Structured Streaming](./01-building-active-data-systems.md) | 2026-03-06 | [Read](https://medium.com/@leelasaikiran4/building-active-data-systems-agentic-ai-meets-spark-structured-streaming-921292fcf298) |
| 2 | [Why My AI Sepsis Detector Failed Until I Stopped Feeding It Raw Data](./02-ai-sepsis-detector.md) | 2026-03-17 | [Read](https://medium.com/@leelasaikiran4/why-my-ai-sepsis-detector-failed-until-i-stopped-feeding-it-raw-data-8f3d6aceaaf2) |
| 3 | [I Gave My Data Pipeline a Contract. It Broke on Day Two. That Was Fine.](./03-data-pipeline-contract.md) | 2026-06-08 | [Read](https://medium.com/@leelasaikiran4/i-gave-my-data-pipeline-a-contract-it-broke-on-day-two-that-was-fine-4f722d2cac7f) |

## What This Series Covers

- Replacing hard-threshold ICU monitors with context-aware AI evaluation
- Spark Structured Streaming + `ai_query()` + Reverse ETL architecture
- Why raw data sent to an LLM fails and how pre-computed features fix it
- Confidence-based routing with asymmetric thresholds for clinical decisions
- Prompt versioning with MLflow
- Schema contracts vs. semantic contracts — and why the gap between them matters

## Related Code

- [icu_deterioration_detection_agentic_streaming](https://github.com/leela56/icu_deterioration_detection_agentic_streaming) — Part 1 pipeline
- [sepsis-risk-dashboard](https://github.com/leela56/sepsis-risk-dashboard) — Parts 2 & 3 pipeline + Next.js dashboard
- Live demo: https://sepsis-risk-dashboard.vercel.app/
