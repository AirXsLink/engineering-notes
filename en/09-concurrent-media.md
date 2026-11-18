---
layout: default
---

# Why One Batch Produced Two Replies: Claiming the Slot Under Concurrency

> The "one reply per discussion" gate was in place. So why did two media items from the same batch each get their own push? Because a read-only gate cannot stop concurrency.
>
> A note on the numbers: the seconds, counts, and test counts below are examples drawn from actual operation. They are not universal benchmarks.

---

**In short:** When a batch arrives together, someone has to claim the slot first. A check that only reads cannot stop concurrency.

---

## 0. The Second Reply Diagnosed Itself

Two nearly identical replies, five seconds apart. The second one began: "Repeated again — the duplicate bug is probably still there."

It was right: the system really did push twice. Both replies carried the same group discussion; only the attached media description differed — a maid outfit in one, blue hair in the other.

The bot had diagnosed its own failure.

---

## 1. One Reply Per Discussion — Why It Leaked

The rule was:

- a discussion is a 90-second window starting at the first message;
- at most one push per window;
- when the window expires, the next discussion starts fresh.

Text messages followed this path cleanly. Media had its own pending and push path, but before pushing it also checked "has this discussion already been answered?"

The problem: that check only read. It never wrote.

---

## 2. Concurrent Arrival: Whoever Finishes Describing First Pushes First

Two media items entered the pending store at 15:30:55. At 15:32:55 they timed out unclaimed, and the system switched to "describe and push directly":

- each item started its own thread;
- each called the describer;
- each ran the pre-push check;
- each pushed.

Descriptions took different amounts of time, so the pushes landed five seconds apart. To the system these were two independent events. To the user it was one topic answered twice.

---

## 3. An Expired Window Means "Never Answered"

The first push was at 15:33:11. The window had been created at 15:30:42; 90 seconds later, at 15:32:12, it expired.

When the window is expired, the "already answered?" check returns False. So both items read "not answered."

Worse: after a media push succeeded, nothing recorded "this discussion has been answered." So the second item's check still saw "not answered."

**A read-only gate is no gate.**

---

## 4. Fix: Claim First, Then Push

Two changes:

1. **Record "answered" after a push** — if the window is expired, open a new one instead of writing nothing;
2. **Claim before pushing** — if already answered, return "no slot"; otherwise claim it immediately, then allow the push.

Now, of N items in one batch, the first claims the slot and pushes; the second falls into "context only." The model still sees it on the next turn, but it no longer triggers a separate reply.

**This is not the same as skipping.** What is skipped is the separate reply, not the event. The media description must stay in context, or the model will later say "I never received the image."

---

## 5. How This Relates to the Previous Note

Note 05 covered F13: **after a discussion has been answered, later media must be blocked.** That was a read gate.

What leaked here was different: **when a batch arrives together, who becomes the first one.** A read gate without a write gate means the first arrival also sees "not answered."

So the fix is not a tighter condition. It is a new action: claim. In a state machine, "check and claim" has to be atomic.

---

## 6. Test the Concurrency

The original 15 boundary tests covered: the second message in the same window is blocked; expiry does not recreate the window; expiry starts a new draw; the media gate; a mention overrides the draw.

All green, and production still leaked — because no test sent **N items in parallel**.

Seven tests were added:

- an expired window first reports "not answered";
- the first item claims the slot;
- the second is blocked;
- after a window, another push is allowed;
- the full push path produces exactly one push;
- both buffers are cleared;
- the second item still enters the group context.

The suite went from 15 to 22.

---

## 7. Three Rules to Take Away

1. **"Check and claim" must be atomic.** Check, then write, with a wait in between, means two concurrent callers both see "unclaimed."
2. **Expiry is not "never happened."** On expiry, open a new window and record it; do not reset the state to "never answered."
3. **Log the skip; keep media in context.** "No separate reply" is not "pretend it never arrived."

---

## 8. Afterword

Concurrency inside one batch is exactly what interval-based tests miss: inside the window and outside the window were both tested; two items arriving together were not.

> If your rate limiter is "check, then write," ask one question: during that wait, what did the other thread see?
