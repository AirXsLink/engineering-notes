---
layout: default
---

# When to Speak in a Group Chat: Deciding to Join

> [Note 05](05-group-burst.md) was about "how many replies per discussion." This one is about the layer before it: whether to join this discussion, whom to answer, and when. It is a design note, not a postmortem of a shipped feature.
>
> A note on the numbers: the probabilities, counts, and seconds below are examples drawn from actual operation. They are not universal benchmarks.

---

**In short:** Probability decides whether a bot pops up. Structure decides whether it feels like a member of the group.

---

## 0. A 0.3 Dice Roll That Lands at the Wrong Time

The first version decided whether to join a discussion by drawing lots: a 0.3 probability per discussion.

After a while, the problem was obvious:

- when the group was quiet, the occasional pop-up felt noisy;
- when the group was busy, the bot went a whole day without saying anything;
- the same kind of message got a reply yesterday and none today — users could not form an expectation.

0.3 was not a wrong number. The mistake was **using a coin flip instead of a judgment**: a group chat already carries signals about whether to speak. They just were not being read.

---

## 1. Three Signals: Relationship, Topic, Mention History

At least three structured signals decide "join or not." None of them needs a model.

**Relationship stage.** Who is speaking, and how close are they: close, trusting, familiar, acquainted, stranger. Someone close speaking is worth answering; a stranger's one-off message can wait.

**Topic heat.** Is the discussion warming up? Several messages around one thing means people are engaged; a single message may just be passing through.

**Mention history.** Has the bot been mentioned, addressed, or asked recently? Being mentioned means someone is waiting for an answer; not being mentioned means it is a bystander.

Together, they answer one question: is this discussion any of my business?

---

## 2. Why Not "Probability Plus Randomness"

A probability model assumes every message is an independent event.

Group chats are not. Several messages form one discussion, with context, a warm-up, and a cool-down. Flipping a coin on independent events produces randomness, not participation.

And 0.3 cannot fit both cases:

- a busy group may produce a dozen messages in 90 seconds; 0.3 means joining every third discussion, which is too noisy;
- a quiet group may have a handful of messages a day; 0.3 means staying silent most of the time, which looks like disappearing.

**The same probability feels opposite in two groups.** The threshold cannot be tuned because what it lacks is input, not a value.

---

## 3. A Decision Table, Not a Score

The design is an explainable decision table, not a weighted score:

| Condition | Decision |
|---|---|
| Directly mentioned | Always reply; no judgment involved |
| Close relationship + topic heating up | Join |
| Mention history + topic related | Join |
| Stranger + single message | Do not join |
| All signals weak | Do not join (silence is the default) |
| Late night / low period | Down-weight, unless mentioned |

The advantage is that every row can be explained in a log line: "joined because the relationship is close and the topic is heating up," not "the draw came up."

The cost is hand-written rules. For a single-machine project, explainability matters more than auto-tuning — when something goes wrong, you need to know which rule fired.

---

## 4. How This Relates to "One Reply Per Discussion"

Two layers, not one switch:

- **Layer one**: should this discussion be joined (this note);
- **Layer two**: once joined, how many replies ([note 05](05-group-burst.md)).

The window and slot in [note 05](05-group-burst.md) prevent flooding; this layer prevents joining at the wrong time. Both are needed before "feels like a member of the group" is even on the table.

Do only layer one and you get "joins the right discussion, three times." Do only layer two and you get "not many replies, but each one lands at the wrong moment."

---

## 5. Implementation Tradeoffs

**Signals must be explainable.** Every decision should record which signals were read, their values, and the conclusion. Otherwise tuning is guesswork and failures cannot be attributed.

**Thresholds must be per group.** A busy group and a quiet group have completely different baselines. One set of thresholds will be wrong for one of them.

**Cold start should be conservative.** With little relationship data, fall back to "always reply when mentioned, otherwise stay silent." Do not use randomness; during cold start, randomness is exactly what users perceive as glitching.

**Relationship is a slow variable.** It changes over weeks, so it should not be the main signal for a single message. It is a weight: for someone close, the same topic heat is worth more.

---

## 6. What Is Not Done, and the Boundaries

- **Topic detection is coarse.** A time window plus keywords decides "is this the same thing," which is inaccurate when several topics run in parallel. Finer detection needs topic clustering; the cost is too high for now.
- **It cannot guarantee "human-like."** It can only guarantee "not random." Humans have many reasons beyond the context — mood, relationship, occasion — and those cannot be fully structured.
- **It will misfire.** A close person saying something unimportant may still get a reply. That is better than never replying, but it is still noise.
- **There is no learning.** The decision table is hand-written and does not learn from feedback. In a single-person project, auditability beats adaptivity.

---

## 7. Three Rules to Take Away

1. **The unit of decision is the discussion, not the message.** Flipping a coin on independent events always yields randomness.
2. **Signals should be structured, explainable, and tunable.** A decision table that can be logged beats a probability that cannot be tuned.
3. **Silence is the default; speaking is the exception.** In a group, saying one sentence too few costs far less than saying one too many.

---

## 8. Afterword

When a bot "speaks at the wrong time," the probability is usually not the problem. It simply never looked at the relationship, the topic, or the mention history.

> If your group bot is "popping up by probability," do not tune that probability first — ask it: what does this discussion have to do with you?
