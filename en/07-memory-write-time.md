---
layout: default
---

# The Write Side of Memory: What to Store, and Whose Fact It Is

> Most agent-memory projects optimize "how to retrieve." Very few ask the cheaper question first: should this ever have been written? This is about five write-side rules and two ways a fact ends up attached to the wrong person.
>
> A note on the numbers: the character counts, line counts, and thresholds below are examples drawn from actual operation. They are not universal benchmarks.

---

**In short:** Raising the retrieval threshold is a patch. Having no write-side rule is the disease.

---

## 0. The Store Grows, Retrieval Gets Noisy

A self-hosted agent's memory store grew from a few dozen entries to a few hundred, and its answers got worse.

The cause was not retrieval logic: the store had swallowed tool output, stack traces, pasted links, questions, and greetings. Retrieval returns by similarity, and noise is just as "similar" as a real fact. So the threshold went up — from 0.40 to 0.55. Less noise, but real facts near the boundary were blocked too.

**The problem was on the write side.** If an item should never have entered the store, a cleverer retriever can only fish it out and throw it away. Every extra piece of noise is another chance for a false recall.

So I added five write rules and two attribution flags. This is the version extracted from real failures.

---

## 1. Rule One: Process Is Not Memory

**Scene.** After an incident, the agent wrote the entire stack trace into memory.

**Why it is wrong.** The trace can be reproduced; what is worth storing is "symptom to root cause." The raw trace takes space, and later it gets retrieved and makes the model believe the error is still happening.

**Rule.** If the text contains `Traceback`, file-and-line markers, `WinError`, or similar, do not write it. Store the conclusion instead.

**A boundary worth noticing.** An error dump usually contains the user's own explanation too. Reject the part that looks like a trace, not the whole message — let the caller decide whether to keep the conclusion.

---

## 2. Rule Two: Volume Is a Signal

**Scene.** One "memory" was 40 lines and 3 KB: a chat transcript.

**Why it is wrong.** A transcript is not a conclusion. It copies the process again, and it outranks the real conclusion by looking more "relevant."

**Rule.** More than 2000 characters or 12 non-empty lines, do not write.

**Why lines as well as characters.** A 5000-character paste and 40 short lines are the same size, but the second has the unmistakable shape of a transcript. Two limits are more stable than one.

---

## 3. Rule Three: Questions Are Not Facts

**Scene.** "What time is the meeting tomorrow afternoon?" was stored.

**Why it is wrong.** A question is a request, not a fact. Remember it, retrieve it later, and the model treats it as something the user cares about — or answers it as if it were known.

**Rule.** Short text (80 characters or less) ending in a question mark, `吗`, or phrases such as "是什么 / 是谁 / 在哪里 / 几点 / 怎么办" is not written.

**Boundary.** Bare particles like `呢` and `么` do not count. The first version treated `呢` as a question ending, so a statement like "我一直都在这里呢" was rejected. Question marks, question particles, and interrogative phrases — not every particle that can end a sentence.

---

## 4. Rule Four: Greetings Are Not Facts

**Scene.** "好的好的" and "嗯嗯" entered the store.

**Rule.** Short text (under 16 characters) made only of greeting characters is not written.

**Why.** It carries no information, but it still gets retrieved and consumes context budget. Worse, it makes "recent memories" always look populated, hiding the fact that nothing real has been written for a long time.

---

## 5. Rule Five: A Link Alone Is Not a Memory

**Rule.** If less than 12 characters remain after removing URLs, do not write.

**Why.** This is the shape of a search result: one link plus "look at this." Store the conclusion, or fetch the content and store it together. A link alone is a pointer that will rot.

---

## 6. Attribution: Wrong Subject Is Harder to See Than Wrong Content

**Scene.** The extractor turned "小雨的妈妈是老师" into "the user is a teacher", and "豆豆特别能吃" into "the user eats a lot".

**Why it is hard to catch.** The stored sentence is perfectly fluent. Retrieval returns it normally. Only a human reading it notices the subject is wrong — and by then it may already have been used in an answer.

Two shapes:

- **relation**: `X 的 <relation noun> ...`. Relation nouns include mother, father, wife, husband, son, daughter, friend, colleague, boss, teacher. The fact belongs to the relation, not the speaker.
- **pet**: a known pet name followed by a verb or adjective. The fact belongs to the pet.

**What to do:** flag, do not rewrite. Rewriting needs the extractor or a model, and a library should not silently change what gets stored. The flag is enough for the caller to keep the subject.

**A negative example.** "我的职业是后端" — "职业" (occupation) is not a relation noun, so nothing should be flagged. Rules need negative tests, or the fix turns "wrong person" into "nothing gets stored."

---

## 7. Fail Open, but Loudly

**Scene (self-inflicted).** After the rules shipped, there was not a single `memory write skipped:` line in the log. It looked like "everything was allowed."

**Real cause.** The plugin directory was missing an `__init__.py`. The import failed, `except` swallowed it, and the gate failed open.

**"The gate allows everything" and "the gate never loaded" look identical in the logs.**

**Fix.** Fail open on internal errors, but always log a warning. Fail-open exists so the main flow is not blocked, not so the failure is silent. After the fix, `memory write skipped: too_short(6)` appeared immediately — that was the proof the gate was actually running.

---

## 8. Extracted Into a Library

The result is `agent-memory-hygiene`, pure functions with no dependencies:

- `assess_write`: five rules, returning `ok`, `reason`, the rule that fired, and the metrics;
- `assess_attribution`: relation / pet flags, without rewriting;
- `safe_assess_write`: fail-open with a warning.

Standard library only, 27 tests, including negative examples such as "我一直都在这里呢" and "我的职业是后端" that must not be rejected.

**Where it deliberately stops.** Importance, semantic deduplication, multilingual rules, and attribution rewriting all need the caller or a model. The library does not pretend to cover them.

---

## 9. Afterword

The value of a write-side rule is not "smarter filtering." It is making an irreversible action cheap: one wrong memory costs many retrievals and a manual check to discover; not writing it costs almost nothing.

> If your agent's memory store is growing, do not touch the retrieval threshold first. Read the last ten entries it wrote.
