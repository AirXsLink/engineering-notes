---
layout: default
---

# Three Layers of Memory: Verbatim, Facts, and Summaries

> With only "verbatim" and "long-term facts," the middle is empty: verbatim is too noisy, facts are too dry, and nobody can answer "where did this fact come from?" This is a design note on three layers, provenance, and multi-route retrieval.
>
> A note on the numbers: the counts, ratios, and parameters below are examples drawn from actual operation. They are not universal benchmarks.

---

**In short:** Memory layers are not about having more layers. Each layer must answer a question the previous one cannot.

---

## 0. A Fact That Cannot Say Where It Came From

The system stores a fact: "the user likes coffee."

Asked "how do you know?", it cannot answer. The fact was extracted months ago from some conversation, and the source was dropped after extraction. The original text still exists in another store, but there is no pointer between the two.

This is not a retrieval problem. It is a **structure** problem: the fact layer keeps the conclusion but not the provenance; the verbatim layer keeps the provenance but is too noisy.

With only these two layers, the middle is a hole.

---

## 1. Two Layers Are Not Enough: Verbatim Is Noisy, Facts Are Dry

The **verbatim layer** records every message word for word: who said what, when. It is trustworthy and traceable; it is also large, repetitive, and noisy. Retrieval by text or embedding easily drowns in "ok" and "sounds good."

The **fact layer** holds conclusions extracted from conversations: one fact per row, with a predicate and a time window. It is searchable and injectable; it is also dry — tone, context, and provenance are gone.

The hard questions live in the middle:

- "What did we talk about last week?" — verbatim is too much, facts are too fragmented;
- "Who brought this up first?" — facts have no source;
- "How did that discussion end?" — verbatim has the answer, but you must read a dozen messages.

**The gap between the two layers is where summaries belong.**

---

## 2. Three Layers: Verbatim, Facts, Summaries

| Layer | Stores | Answers | Cost |
|---|---|---|---|
| Verbatim | exact messages + time + speaker | "What were the exact words?" | large, noisy |
| Facts | extracted conclusions + predicate + time window | "What does the user like?" | dry, no provenance |
| Summaries | a discussion compressed into a topic + key conclusions | "What did we talk about last week?" | drifts, costs tokens |

These are three granularities, not three copies. The design requirement is that they point at each other:

- fact → verbatim: each fact can return to the sentence(s) it was extracted from;
- summary → fact/verbatim: each summary points at the discussion it compressed;
- verbatim → summary: any message can be looked up to see which summary absorbed it.

Without these pointers, the three layers are three islands.

---

## 3. Provenance: Upgrading the `source` Field

The `source` field used to hold a "channel label": which entry point the memory came from. Useful, but it cannot answer "which sentence?"

The upgrade is simple: replace the channel name with a **verbatim ID** (or a list of IDs). Channel information moves to a separate field; the two are not mixed.

Three capabilities open up:

- **Explainability**: when the model says "the user likes coffee," the original sentence can be opened;
- **Correction**: when verbatim text is deleted or expires, the fact can be found through the pointer and handled together;
- **Audit**: the distribution of a fact's sources shows which conversations the extractor handled well.

The cost is one extra field on write and the need for the extractor to receive the verbatim ID. For existing data, it is a backfill.

---

## 4. Multi-Route Retrieval: BM25 + Embeddings + Entities

Each route has a blind spot:

- **BM25 (lexical)**: strong on proper nouns, names, and abbreviations; weak on paraphrases;
- **Embeddings (semantic)**: strong on paraphrases; weak on exact names;
- **Entities (people / topics)**: strong on "who and who"; needs normalization.

The benefit of running three routes is not "more accurate." It is **complementary**: one lexical hit, one semantic hit, one entity hit together are steadier than any single route.

The cost is ranking. The scores are not comparable: BM25 is unbounded, cosine is 0–1, entity matches are boolean or counts.

---

## 5. RRF: Rank, Do Not Weight

The cheapest fusion is **Reciprocal Rank Fusion**:

```
score(d) = Σ_i  1 / (k + rank_i(d))
```

- Each route contributes only "where this result ranked," not its raw score;
- `k` is usually 60; it dampens the head gap so items appearing in multiple routes are steadier;
- No per-route weights, no score normalization.

It is insensitive to "one route much stronger than another" — exactly what is wanted here, because the routes differ in quality and tuning weights would add guesswork.

**RRF is not the best fusion; it is the fusion that works without tuning.** For a single-machine project, that tradeoff pays.

---

## 6. Why Not a Graph Database

Some peer systems use a temporal property graph: entity nodes, fact edges, bi-temporal time, and graph traversal at retrieval. It works, at a cost:

- it needs a graph database or extension; a single-machine 8 GB project cannot afford it;
- at small data sizes, a one-hop traversal is not much better than two SQL joins;
- operations get heavier: backup, migration, and debugging all grow.

The substitute is two time columns: `valid_from` / `valid_to`. They answer "when was this fact valid," covering the most common graph query. Graph traversal is deferred — **until the data really needs it**.

---

## 7. Tradeoffs and Boundaries

- **Summaries drift.** The same discussion compressed at different times may yield different summaries. Mitigation: store only "topic + key conclusions + verbatim IDs," never free prose.
- **Backfill costs.** Moving `source` from channel names to verbatim IDs means remapping history; unmatched rows stay empty and flagged.
- **Entity normalization is the hardest part.** One person has nicknames, abbreviations, and typos; one work has Chinese and English titles. Done poorly, the entity route treats noise as signal.
- **RRF's `k` is not magic.** 60 is a common default, not an optimum; recheck it on a new dataset.
- **More layers are not better.** Each layer adds pointers and a lifecycle; if the gain is small, do not add it.

---

## 8. Three Rules to Take Away

1. **Each layer must answer a question the previous one cannot.** If it cannot, do not add it — a layer is maintenance cost.
2. **Every fact must have provenance.** A fact that cannot be traced is noise the moment it is challenged.
3. **Fuse ranks, not scores.** When route scores are not comparable, RRF is far cheaper than tuning weights.

---

## 9. Afterword

The depth of a memory system is not "how much it remembers." It is whether, when asked, it can say clearly: where this sentence came from, when it was valid, and how it relates to the other facts.

> If your memory store cannot answer "how do you know?", do not add a vector database first — add the provenance.
