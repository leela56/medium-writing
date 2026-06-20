# I Rechunked Once. It Beat Three Embedding Model Swaps.

**Published:** 2026-06-16
**Medium:** https://medium.com/@leelasaikiran4/i-rechunked-once-it-beat-three-embedding-model-swaps-f902f320948d
**Read Time:** ~8 min
**Tags:** rag, llm, embeddings, chunking, clinical-ai

---

*Cosine similarity can't fix a chunk boundary that split the answer.*

The query was "what beta-blocker was this patient discharged on." The top result, with a cosine score north of 0.83, was a chunk from the History of Present Illness section that said the patient had been taking metoprolol at home prior to admission. The Discharge Medications section — which actually answered the question and was three pages later in the same document — did not appear in the top five. Same document in the index. Same embedding model. Wrong chunk.

This is not a retrieval problem. The retriever did exactly what it was asked to do: find the chunks whose vectors were closest to the query vector. The chunk it found was semantically close to the question. It was also wrong, because the chunking strategy had stripped away the structural signal that would have told a human reader which part of the document to read.

I had spent three weeks before this swapping embedding models. The embedding swaps moved recall@5 around four points. **The rechunking moved it nineteen.**

## The thesis nobody wants because it's boring

Everyone is tuning embedding models. Everyone is adding rerankers. The MTEB leaderboard moves by tenths of a point and people change their stack over it.

The actual bottleneck in most production RAG systems is upstream of retrieval. It's that the chunk, as a unit, is wrong for the data. The 512-token sliding window with 50-token overlap is a default that came out of papers written on clean web text and got copied into every tutorial. When the corpus is anything with internal structure — discharge summaries, contracts, SEC filings, internal wikis with consistent templates — that default cuts answers in half and there is no embedding model on earth that fixes it. Reranking doesn't fix it either, because the right chunk isn't in the candidate set to be reranked.

**The embedding leaderboards are measuring the wrong thing for most of the corpora people are actually trying to put into production.**

## What I actually did

The corpus was around 40,000 discharge summaries. The first version of the pipeline used the LangChain `RecursiveCharacterTextSplitter` with chunk size 512 and overlap 50 — the default that every RAG tutorial reaches for and which is correct for almost no real corpus.

Discharge summaries have a prior. They have sections. The sections have names that vary across institutions but are consistent within an institution: "Hospital Course," "Discharge Medications," "Discharge Disposition." These are not paragraph breaks. They are semantic boundaries. The medication a patient was discharged on is in the Discharge Medications section and almost never elsewhere. A chunker that doesn't know about these boundaries will, statistically, cut across them.

What I switched to was structural chunking driven by a regex pass over the known section headers for the EMR template the documents came from. Each section became a chunk. Sections longer than about 1200 tokens got sub-chunked, but only within themselves, never across a section boundary. Section name went into the metadata. The retriever could now filter or boost on section if the query had section-relevant terms.

**Results on a held-out eval set of ~400 clinician-written queries:**
- Recall@5: 0.62 → 0.81
- Mean Reciprocal Rank: 0.41 → 0.67
- Latency: unchanged
- Index size: slightly smaller (structural chunks deduplicated boilerplate the sliding window had been replicating)

I want to be honest about how unprincipled the regex pass is. The section headers in clinical documents are not standardized. The same hospital uses two different EMR templates depending on the unit. "Discharge Medications" in one, "Medications at Discharge" in another, and a legacy template still in use that just says "MEDS" in all caps. The regex is a list of patterns I maintain by hand. It will be wrong for any institution I haven't seen.

This felt like a workaround at first. **It's actually just the work.** There is no general structural parser for clinical documents because clinical documents are not generally structured.

## The part about embedding models

Three swaps, same corpus, same (bad) chunking, same eval set:

```
text-embedding-3-small   →  recall@5 = 0.62
text-embedding-3-large   →  recall@5 = 0.64
BGE-large-en-v1.5        →  recall@5 = 0.66
```

Four points of recall across three models. Then I changed the chunker and recall went to 0.81 with the cheapest model in the list. The same rechunking applied to BGE got to 0.83.

The model gap is real. It is also tiny compared to the gap that opens up when you fix the unit you're embedding.

I want to acknowledge that this comparison is unfair to the embedding models — they were doing their job. Their job was finding chunks similar to the query, and they were finding chunks that were similar. **The system was failing one layer up. Tuning a layer that isn't broken is the most expensive kind of work, because it feels productive and the numbers move just enough to keep you doing it.**

pgvector was fine through all of this. At 40,000 documents with 250,000 chunks, pgvector with an HNSW index served queries in under 30 milliseconds and I never thought about it again. The vector database is not the bottleneck.

## What I still don't have answers for

**Nursing notes.** These are written as a single block of prose, no headers, no consistent structure, sometimes literally one paragraph that runs for two pages. Structural chunking has nothing to attach to. I'm back to token windows for this subset and the recall is bad. I've tried semantic chunking (using embedding similarity between sentences to find natural break points) — marginally better than fixed-size, substantially slower, and I haven't decided if the tradeoff is worth it.

**Cross-section queries.** What happens when a query needs the diagnosis from one section and the medication from another to answer correctly? Structural chunking made this harder, not easier. I'm hacking around it with a second-pass retrieval that filters by document ID once a relevant chunk is found, then pulls adjacent sections. It works. It doesn't generalize.

**The section header regex maintenance tax.** Every new institution onboarded means updating the regex by hand. I've thought about training a small section classifier on token-level labels. I think it would work. I haven't done it because the regex is good enough and the labels would take a week to produce.

## The lesson, such as it is

The embedding model is rarely your problem. The vector database is almost never your problem. The chunker is almost always your problem, and the reason nobody talks about it is that fixing it requires actually understanding your corpus — unglamorous work that doesn't generate a leaderboard.

If your retrieval is bad, try rechunking before you swap embedding models. The intervention is cheaper and the effect is usually larger. You'll also learn something about your data in the process, which is not true of an embedding swap.

---

*If you've shipped RAG on a corpus where the documents have real internal structure and you have a chunking approach that's better than maintaining a regex by hand, I'd actually like to hear how you got there.*
