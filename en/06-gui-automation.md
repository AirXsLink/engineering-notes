---
layout: default
---

# Why "Sent Successfully" Means Nothing in GUI Automation

> The most dangerous signal in keyboard automation is a success return.
>
> A note on the numbers: the thresholds, timeouts, failure counts, and send counts below are examples drawn from actual operation. They are not universal benchmarks.

**In short:** A GUI automation success only proves a key was pressed. Only the message store proves delivery.

---

## 0. A "Sent" Message That Nobody Received

The log said `[UIA✓] sent` (UIA is UI Automation, the Windows accessibility API). The cursor moved, the enter key was pressed, and the program returned success.

The user received nothing.

Checking the message store revealed that the message had never landed. The keystrokes had gone somewhere — a search box, another conversation, or into the input method.

This is the fundamental problem of GUI automation. Every success signal it can observe proves only that a key was pressed. None of them proves that the message reached the right person.

What follows is what a production chat bot learned the hard way, and the three defenses that finally made sending reliable.

---

## 1. Defense One: Before Sending, Confirm the Window

The most common GUI automation mistake is assuming the current window is the target conversation.

So before sending, read the window title and compare it with the expected contact name:

- Run OCR (optical character recognition) on the title;
- Compute similarity against the expected name;
- If below the threshold, switch the conversation and read again;
- If it still does not match, **abort the send.** Never send blind.

Two parameters matter:

- **Threshold 0.85.** Too loose sends to the wrong person; too tight causes constant re-switching. 0.85 was calibrated on a batch of real titles.
- **Loose matching as a fallback.** Beyond strict comparison, allow "containment plus a sufficiently long common substring" to absorb OCR jitter.

### The trap: the tolerance threshold blocked the shortest name

The loose-matching path required the expected name to contain at least four CJK (Chinese, Japanese, Korean) characters.

The owner's display name had only two. **The tolerance logic never ran.** The check fell through to strict matching, similarity 0.75 failed, and the bot paused replies to that contact for 15 minutes after three attempts.

OCR had also read one character as `%`.

The fix was a whitelist: for registered names only, admit if both fragments appear. Add a negative test so a similar-looking name is rejected.

**Lesson:** when you put a threshold under a tolerance path, ask who falls below it. The shortest and rarest names fall first.

---

## 2. Defense Two: Focus and Epoch, During Sending

Before pressing a key, confirm three things:

1. **The foreground window is the target application** — not a browser, not an editor.
2. **The conversation is still the one you started with** — nobody switched it manually.
3. **No concurrent send is fighting for the same window.**

The implementation:

- Check the foreground process before typing, before pasting, and before pressing Enter.
- Use a process-level send lock so only one message is written at a time.
- Record a conversation epoch at the start, read it again before Enter, and abort plus clear the draft if it changed.

**Why check again before Enter:** there is a time gap between "paste finished" and "press Enter." In that gap, another person or thread can switch the window. Clearing the draft is mandatory. Aborting while leaving half a message in the input box means the next send carries it along.

---

## 3. Defense Three: After Sending, Query the Message Store

After a send, ask the message store: "Did this exact message really land?"

This is the most expensive and most valuable step:

- Per-message verification timeout: 60 seconds;
- On failure, recheck once after 15 seconds to rule out sync delay;
- Only then switch back and resend.

### The trap: too short a verification window causes duplicate sends

The earliest window was 20 seconds. Message sync sometimes took 31 seconds, so the check reported failure, the bot resent, and the user received two copies.

After widening the window to **300 seconds** and increasing the check from 5 to 20 messages, duplicate sends disappeared.

**Lesson:** the verification window must cover the P99 of sync delay, not the average. Too short is worse than no verification: it reliably manufactures duplicates.

### The other trap: UIA says success, the message never lands

In one case the log showed `[UIA✓]`, verification timed out, and the bot switched back and resent. The message store ended up with one message: the first one had never landed.

"Key press succeeded" and "message landed" are two different facts, and the gap can open anywhere — input method, focus, window repaint. **Only the message store is truth.**

---

## 4. Circuit Breaker: Stop Getting It Wrong

When failures repeat, more retries only produce more wrong messages. So break the circuit:

- Three pre-send failures for one contact in 15 minutes → pause sends to that contact;
- Consecutive global failures → pause for 10 minutes;
- During the pause, log only; resume automatically when it ends.

The breaker is not punishment. It creates **a window for a human to intervene.** Without it, automation keeps running fast in the wrong direction.

---

## 5. Symbol-Only Nicknames

Some display names are pure symbols: "。。。", "---". OCR returns symbols, so similarity is useless. Worse, when switching conversations by "searching the nickname," those symbols can be typed into the search box.

The fix is **shape comparison**: if the character count differs by at most one and there are no CJK characters, treat the shape as matching and use a different confirmation path.

The goal here is not accuracy. It is **not doing the wrong thing.** Abandoning one send is better than typing a row of symbols into a search box.

---

## 6. Five Principles

1. **A GUI return value is not a fact; it is an attempt.** Trust external state — the message store, the database, the queue.
2. **Before, during, and after are all required.** Before prevents the wrong recipient; during prevents a mid-send switch; after catches "sent but never arrived."
3. **Retries must be idempotent.** Verify before resending; never resend blind. Otherwise one hiccup becomes two messages.
4. **A circuit breaker protects against repeating the mistake.** When failures repeat, stopping is safer than continuing.
5. **Every tolerance path needs negative tests.** Whitelists, fuzzy matching, and shape comparison each need a "must not pass" case.

---

## 7. Afterword

Reliability in GUI automation comes down to one sentence: **trust no single signal.**

Do not trust the window title, the focus, the send return value, or a single verification. Each layer needs its own check, and if any layer says no, stop.

After this shipped: 98 consecutive sends, 0 pre-send failures, 0 verification failures, 0 circuit breaks. Not luck — every message passed three gates.

> If your automation script is "pretending to succeed," add post-send verification first. It will immediately tell you how much you have been losing.
