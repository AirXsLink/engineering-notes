---
layout: default
---

# One Topic, One Reply: Burst Control for Group Chat Bots

> Staying silent is not the worst thing a group chat bot can do. Answering the same topic four times is. This article is about treating messages as a conversation instead of events.
>
> A note on the numbers: the seconds, message counts, and test counts below are examples drawn from actual operation. They are not universal benchmarks.

**In short:** Move the unit of decision from "message" to "discussion," and the reply count stops tracking the message count.

---

## 0. Forty Seconds, Four Replies

Between 17:16 and 17:18, a group saw this:

- 17:16:37 — someone posted a link;
- 17:17:13 — someone mentioned the bot: "What do you think?";
- 17:17:18 — another person sent an emoji;
- 17:18:07 — someone added a complaint.

The bot replied four times. Three of the replies discussed the same link. Someone asked, "What's wrong with her today?"

Each reply had a reason: a mention must be answered, an emoji must be acknowledged, a new message must be handled. Together, they turned one discussion into four independent events.

**The question for a group bot is not "reply or not." It is "what counts as one thing."**

---

## 1. Define the Event First

In a direct message, one message is one event. In a group, it is not. Several people talk around one topic within a minute; together, those messages form **one discussion**.

So the unit of decision moves up:

- A discussion is a time window starting at the first message (90 seconds here);
- At most one reply is pushed per window;
- When the window ends, the next discussion starts fresh.

The direct consequence: **reply count no longer scales with message count.** Ten messages in a group may produce at most one reply.

---

## 2. Draw Before Replying

"At most one reply" does not mean "always one reply." In a random-engagement mode, the system first decides whether to join the discussion at all.

The flow:

1. A new discussion starts; draw once by probability;
2. If selected, the window may produce one reply;
3. If not selected, stay out of the whole discussion;
4. If mentioned, **always reply — no draw, and no slot consumed.**

The fourth rule matters. A mention is an explicit request, so probability must not apply. But "always reply" must not become "reply more": it guarantees that one reply happens, not that an extra slot opens.

---

## 3. Three Message Types, Three Gates

Group messages are not all text. Images and emoji have their own rhythm. They cannot share the text gate, but they must not bypass it entirely.

The design:

| Message type | Consumes the one-reply slot? | Notes |
|---|---|---|
| Text | Yes | Main path: draw + window |
| Image / emoji (caption available) | No | Has its own pending and push logic |
| Image / emoji (discussion already answered) | No push; context only | Prevents repeated answers |

The third row was added later, after a production incident (next section).

---

## 4. Three Seams That Produced Four Replies

Replaying the four replies revealed three overlapping seams.

**Seam 1: mentions always reply.** By design, not a bug.

**Seam 2: emoji pushes did not consume the slot.** They had an independent push path, so the "one per discussion" rule did not apply. But the payload still contained **the entire discussion plus a "respond to the specific event" instruction.** The model saw the full context and re-answered the link it had already answered 40 seconds earlier.

**Seam 3: the window quietly recreated itself after expiry.** The window started at the first message and lasted 90 seconds. A message could be received inside the window but only claim the slot after a 5-second buffer — by then the window had expired. The code saw the expiry, created a new window, reset the slot, and let another reply through.

Each seam looked reasonable alone. Together, they produced four replies to one discussion.

---

## 5. Fix: Expiry Means No Slot

Three changes:

1. **An expired window returns "no slot."** Never recreate it in place. The next message starts a new draw.
2. **If the discussion has already been answered, images and emoji enter the context only.** Skipping means "no separate reply," not "pretend it never happened." The media description must stay in context; otherwise the model will later say "I never received the image."
3. **Log the skip:** `discussion already answered; media added to context only`. A silent skip is indistinguishable from a silent failure.

---

## 6. Test Boundaries, Not Intervals

The logic has 15 tests, three of which target boundaries:

- second message in the same window is blocked;
- expiry does **not** recreate the window;
- expiry starts a new draw;
- emoji gate: in an answered discussion, emoji does not trigger a reply;
- mention override: no draw, but mentioned, so reply.

"Inside" and "outside" are calm intervals. Bugs live at "just expired."

---

## 7. Three Principles

1. **The unit of real-time in a group is the discussion, not the message.** Move the decision unit up one layer, and reply count decouples from message count.
2. **Separate "must reply" from "draw."** Mentions solve "don't miss"; the draw solves "don't be noisy." One switch cannot do both.
3. **Log the skip; keep media in context.** "No separate reply" is not "nothing happened." The model still needs to know what the user sent.

---

## 8. Afterword

After this shipped, "one topic, many replies" largely disappeared. The remaining multi-reply cases are both by design: a mention always gets a reply, and an expired window starts a new draw.

A group bot is mature not when it replies fast, but when it **knows it is participating in the same discussion.**

> If your bot answers one question four times, do not tune the prompt first. Look at how it handles the window boundary.
