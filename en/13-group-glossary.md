---
layout: default
---

# A Group Glossary: Understanding Slang You Can Read but Not Parse

> The most distinctive layer in a group chat is not long sentences. It is slang: abbreviations, nicknames, and memes only this group understands. Verbatim recall cannot solve it. This is a design note on a per-group glossary.
>
> A note on the numbers: the frequencies, counts, and budgets below are examples drawn from actual operation. They are not universal benchmarks.

---

**In short:** Slang is not unfamiliar vocabulary. It is a group-local convention, and it needs a per-group table that grows by frequency.

---

## 0. It Read Every Character and Still Missed the Joke

Someone in the group posted: "again with 牢大."

The bot replied: "Who is 牢大?" It recognized both characters, and it even retrieved the message from the verbatim store. But it did not know that in this group, the phrase is a fixed way of referring to a particular person or thing.

This is not a model intelligence problem. The word has a **conventional meaning inside this group**, and nothing in the system records that convention.

Verbatim recall can find who said the word. It cannot find what the word means here.

---

## 1. Why Retrieval Misses Slang

The three usual routes each have a blind spot:

- **Lexical (BM25)**: the surface form of slang is unrelated to its meaning — 牢大 is not 老大;
- **Embeddings (semantic)**: the model has no representation for a group-local convention, so similarity does not help;
- **Entities (people / topics)**: slang is precisely the thing that is not a standard entity, so normalization filters it out.

Slang needs a **separate table**, not a hope that the retriever will figure it out.

---

## 2. The Table: Term / Meaning / Count / Time

The minimum structure:

```
glossary(
  chat_id,      -- per-group isolation
  term,         -- the slang itself
  meaning,      -- meaning inside this group
  count,        -- occurrences
  confidence,   -- auto-derived is lower than human-confirmed
  first_seen,
  last_seen,
  source        -- automatic / confirmed
)
```

Design points:

- **Per-group isolation**: the same term can mean different, even conflicting, things in two groups;
- **Count is a signal**: a single occurrence is likely a typo or a one-off joke; several occurrences are worth a record;
- **Confidence tiers**: auto-derived and human-confirmed meanings must not carry the same weight, and injection priority differs;
- **Time fields**: slang expires. Last year's meme may be dead this year.

---

## 3. Inferring Meaning: Frequency, Co-occurrence, Confirmation

Automatic inference does not aim for "fully automatic." It aims for "low risk":

**Frequency threshold.** A term enters the candidate list only after appearing at least N times in the group. Too small a threshold pulls in typos; too large a threshold never creates an entry. Experience puts N between 3 and 5, adjusted by group activity.

**Co-occurrence.** Look at what it appears with:
- always alongside one person's name → possibly that person's nickname;
- always inside one topic → possibly that topic's shorthand;
- always paired with an emotional word → possibly an evaluative meme.

**Human confirmation.** Auto-derived meanings are low confidence and go to a review list; one confirmation promotes them to high confidence.

**Why not fully automatic**: misunderstanding slang is worse than not understanding it — explaining 牢大 as "the boss in prison" would wreck the reply.

---

## 4. Injection: Only Hits, With a Budget

Once the table exists, never inject the whole thing — a group may have hundreds of entries, and a full dump would crowd out real content.

The approach is **match per message**:

1. Which entries appear in the current message;
2. Take only the hits with sufficient confidence;
3. Inject at most K (say 3), sorted by hits × confidence;
4. Keep the format short: "term: meaning," no provenance.

Matched entries should also feed the "should we speak" decision: when a meme only insiders know shows up, the discussion is warming up inside the group, and joining is usually more appropriate than usual.

---

## 5. Per-Group Isolation and Contamination

The glossary is naturally per-group, but three implementation traps are easy to hit:

- **Cross-group bleed**: group A's slang injected into group B's reply, where nobody understands it;
- **Private-chat contamination**: using group slang in a one-on-one chat feels equally wrong;
- **Malicious entries**: someone may deliberately teach an offensive "meaning"; automatic inference must not trust it.

Constraints: every query carries `chat_id`; private chats do not query the group table; low-confidence entries never reach injection, only the review list; offensive entries are discarded.

---

## 6. Boundaries and Non-Goals

- **No fully automatic semantics.** Slang meaning depends on group consensus; inference can only reach "candidate," and a human confirms.
- **No cross-group sharing.** The same term differs by group; a shared table only creates conflicts.
- **No real-time learning.** Inferring meaning on every new occurrence is costly and noisy; batch inference, say daily, is steadier.
- **No full-table injection.** It crowds the context, and most entries are irrelevant to the current message.
- **No guarantee of "human-like."** Using slang correctly is a bonus; using it wrongly is a penalty. Better to skip than to misuse.

---

## 7. Three Rules to Take Away

1. **Slang is a group-local convention, not unfamiliar vocabulary.** It needs a per-group table, not a bigger model.
2. **Infer conservatively; inject sparingly.** Misunderstanding is worse than not understanding; full-table injection is worse than none.
3. **Tables must expire.** Slang has a lifecycle; entries whose `last_seen` is too old should drop out of injection instead of staying forever.

---

## 8. Afterword

Whether a group bot "feels like a member" depends heavily on whether it understands the group's slang. And slang is exactly what general models are worst at — because it is not a language problem, it is a **convention** problem.

> If your group bot keeps asking "what does that mean?", do not swap the model first — build it a group glossary.
