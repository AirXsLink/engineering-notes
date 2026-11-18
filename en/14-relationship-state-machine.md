---
layout: default
---

# Relationship as a State Machine: Eight Dimensions, Six Stages, and Proposals

> "We're close" is an unexecutable sentence: it cannot drive tone, cannot explain a boundary violation, and cannot be rolled back. This is about turning a relationship into a state machine that is computable, auditable, and reversible.
>
> A note on the numbers: the dimensions, stages, and thresholds below are examples drawn from actual operation. They are not universal benchmarks; specific values and identities have been redacted.

---

**In short:** A relationship is not a description. It is a set of states that drive behavior, can be audited, and can be rolled back.

---

## 0. "We're Close" Cannot Be Executed

The first version kept the relationship as one line in the persona: "very close to the owner."

It drove nothing:

- How warm should a reply be? "Very close" gives tone no scale;
- The person asks a boundary-crossing question — answer or not? "Very close" gives the boundary no criterion;
- The relationship sours one day — how is that recorded? There is no state, only a sentence, and editing it leaves no history.

**A description is not a state.** A state must answer three questions: what is it now, why did it become this, and how do we go back.

---

## 1. Why Eight Dimensions Instead of One "Affinity"

The intuitive design is a single 0–1 affinity score. It conflates different kinds of relationships:

- high trust, low affection: willing to tell the truth, but polite and distant;
- high affection, low trust: warm, but no secrets;
- high respect, low attraction: takes advice seriously, but finds the person boring.

One number cannot express those differences. So it becomes eight dimensions:

| Dimension | Answers |
|---|---|
| Familiarity | How well do we know each other? Can I use inside references? |
| Trust | Can I share secrets and speak plainly? |
| Affection | Do I want to start conversations? |
| Boundary comfort | How far can a joke go? |
| Respect | Is their judgment worth listening to? |
| Rapport | Can I catch their references and shared history? |
| Attraction | Do they have competence or contrast? |
| Dependence | Are they clingy? Should I push them to stand on their own? |

These are not a psychological scale. They are an **engineering approximation**: every dimension must map to an observable behavior (tone, initiative, refusal), or it should not exist.

---

## 2. Six Stages: The Stage Decides How Far Speech Can Go

Dimensions are continuous; stages are discrete:

```
stranger → acquainted → familiar → trusting → close → bonded
```

A stage is not "the score reached X, so jump." It is **threshold plus adjacent promotion**:

- each stage has admission conditions (several dimensions reach thresholds together);
- promotion only to the adjacent stage, never skipping;
- skipping is a bug, not a surprise — going from "stranger" to "close" means all the evidence in between was lost.

The stage **constrains tone**: at a low stage, do not pretend familiarity or say "we used to..."; at a high stage, teasing and shared history are allowed. Dimensions decide how warm; the stage decides whether warm is permitted.

---

## 3. How Values Move: Events, Not Time

Relationship values must not rise with time — otherwise idling would build closeness. Increments must be triggered by events:

- being helped or listened to → trust and affection rise slightly;
- conflict then reconciliation → trust may dip then recover, boundary comfort rises;
- sharing something private → familiarity and rapport rise, if it was voluntary;
- being corrected or refused → respect rises if handled well, affection may fall.

Every change carries three things: **event, delta, evidence**. Evidence is the specific message that triggered it; a change without evidence is not written.

Time does exactly one thing: **decay** (section 6).

---

## 4. Proposal Review: The Model Does Not Move the Relationship Itself

Letting the model write relationship values directly creates two problems: it may inflate closeness to please the user, and it may misread one emotional exchange.

So changes go through proposals:

1. The model or a worker notices an interaction worth recording and produces a proposal: which dimension, how much, with what evidence;
2. The proposal enters a review list and does not take effect;
3. After a human (or a deterministic rule) approves, it is written;
4. The write is logged: time, dimension, old value, new value, evidence, approver.

**The value of proposals is not accuracy. It is reversibility.** A misjudgment can be seen, rejected, and rolled back; a direct write leaves no trace.

---

## 5. Boundary Rules: The Stage Decides What Cannot Be Said

The most practical use of relationship state is judging boundary crossings.

The same sentence means different things at different stages:

- low stage: "how much do you earn?" → crossing; refuse and redirect;
- high stage: the same question → answer it, or deflect with a joke;
- low stage: "I miss you" → pressure; cool down;
- high stage: the same words → can be received.

The implementation is a triplet: detect the boundary signal, choose a response by stage, record the crossing. The key is **near-zero false positives** — misreading ordinary care as a crossing hurts more than missing one.

---

## 6. Forgetting Curve: Values Decay, Stages Do Not

After a long silence, the relationship should fade, but not reset.

Two layers:

- **values decay slowly** by half-life;
- **stages never auto-demote**: values may fall back into the previous stage's range, but the stage stays.

Why no auto-demotion: a user who returns after a while and finds the relationship reset to zero has the worst possible experience. A stage represents trust that was once built; it should only drop on explicit events (conflict, blocking), not on time.

---

## 7. Why Not Let an LLM Judge the Relationship Directly

LLMs are good at judging "is this sentence warm?" They are bad at being the state:

- **inconsistent**: the same input reads close today and neutral tomorrow;
- **unauditable**: cannot explain why it changed;
- **irreversible**: once written, there is no history;
- **easily steered**: a user saying "aren't we good friends?" may push it along.

The compromise: **LLM proposes, state machine decides.** The LLM reads conversations for interactions worth recording; the state machine owns values, stages, review, and rollback.

---

## 8. Boundaries and Non-Goals

- **No automatic promotion.** Stage changes need approval, or one misjudgment permanently raises the risk.
- **No cross-platform sharing.** The same account behaves differently on different platforms; merging only contaminates.
- **No mind reading.** Infer from behavior; if it cannot be inferred, keep the old value; no evidence, no write.
- **Values are not psychological measurement.** They are an engineering approximation for driving behavior, not labels for people.
- **No relationship ranking.** Eight dimensions are independent per contact, not a leaderboard of who matters more.

---

## 9. Three Rules to Take Away

1. **A state must drive behavior.** A dimension that changes nothing should not exist.
2. **Changes need evidence, review, and a log.** Reversibility beats accuracy.
3. **Time only decays; it never promotes.** Relationships accumulate through events, not idling.

---

## 10. Afterword

After relationship became a state machine, the biggest change was not "more human." It was **explainability**: why the tone was warm today, why that question was refused, why the relationship dropped a stage last week — all of it is on record.

> If your agent also claims to be "close to the user," ask it one question: what is that "close" in the code?
