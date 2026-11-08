---
layout: default
---

# Six Fallback Decisions for Multimodal Chatbots

> What to do when an image is not ready, not readable, or not there at all.
>
> A note on the numbers: all seconds, character counts, and pixel thresholds below are examples drawn from actual operation. They are not universal benchmarks.

**In short:** The hard part of multimodal fallback is not seeing. It is answering when you cannot see.

---

## 0. "It Didn't See It, Then It Did"

At 18:56, someone sent an image to a group.

The bot replied: "This image is empty. I can't see anything."
Forty seconds later, it described the contents.
People started asking, "What's wrong with her?"

The model was not the problem; the **decision** was. The caption was not ready, so the system pushed "unrecognized"; when the caption arrived, it pushed again. The user saw "it didn't see it, then it did." The system saw two independent messages.

The hard part of a multimodal pipeline is never "can the model see." It is **what to answer before the model sees, after it fails to see, and when there is no image at all.**

These are six decisions I actually made in production.

---

## Decision 1: Wait or push?

**Situation.** A user sends an image. Captioning takes 15–25 seconds. The message buffer is 5 seconds.

**Options:**
- Push on schedule with "content not recognized," then send the caption later.
- Defer until the caption is ready or the deadline passes.

**Choice:** defer, with a hard cap.

When the buffer expires and a caption is still in flight, recheck every 2 seconds, for at most 90 seconds. After 90 seconds, stop waiting and push without media.

**Why:** The first option creates "it didn't see it, then it did." The context receives two contradictory statements about the same image, the model answers the first one, and the user experiences a split personality. The second option costs latency, so it needs a ceiling; otherwise a dead caption service blocks the reply forever.

**Cost:** worst case, one message is delayed by 90 seconds. That is a ceiling, not the norm. Captioning usually finishes within 20 seconds.

---

## Decision 2: Retry or accept?

**Situation.** A screenshot with text. The model returns "21:30" — it read the status bar, not the body.

**Options:**
- Accept the result.
- Retry once with a more direct transcription prompt and keep the longer output.

**Choice:** for long images and screenshots, retry when the caption is shorter than 40 characters.

**Why 40:** it is an empirical threshold, not a scientific constant. A very short result usually means the model grabbed peripheral information — a status bar, a title, a timestamp — instead of the body. Forty characters is roughly the lower bound of a one-sentence useful description.

**Why only long images:** a normal photo may legitimately have a short description ("a cat on a sofa"). Retrying those wastes an inference and may make a simple description worse. The retry condition must include "text-heavy image."

**Cost:** one extra retry costs 10–20 seconds, but only for long images.

---

## Decision 3: Describe or transcribe?

**Situation.** A 646×1400 chat screenshot. The model returns "a phone screenshot with chat messages." All the text was summarized away.

**Options:**
- Always describe the scene.
- Switch long images to line-by-line transcription.

**Choice:** if the long edge is at least 700 pixels, switch to transcription, with a 1400-pixel long edge and 900 output tokens.

**Effect:** on the same screenshot, the caption went from 84 to 547 characters, and the key lines were preserved.

**Why 700:** it is the empirical line between "phone screenshot" and "ordinary photo." Portrait screenshots usually exceed a thousand pixels on the long edge; transcribing an ordinary photo makes no sense even if it is tall.

**Cost:** transcription is slower and more token-hungry, and it produces noise on images with no text. Provide both `auto` (decide by size) and `transcribe` (force), so the caller can override.

---

## Decision 4: Placeholder or drop?

**Situation.** The model returns a single "□" — the prompt said "use □ for characters you cannot read," and it complied.

**Options:**
- Treat "□" as a valid caption and inject it.
- Mark it invalid and replace it with a placeholder.

**Choice:** mark it invalid and replace it with "content not recognized."

Rule: after removing punctuation, fewer than three meaningful characters means invalid.

Two more rules came later: placeholder density (four or more `□`-style characters, more than twice the meaningful characters) and prompt echo (two or more instruction phrases copied back). A too-short transcription is no longer retried as transcription; the adapter asks for a plain description instead. All three rules came from real misfires.

**Why not drop it:** dropping the image and using a placeholder are different. If you drop the image, the model believes no image was sent. A placeholder at least preserves the fact that an image arrived and could not be read.

**The more important step:** apply the check at the **injection point** as well, not only at the generation point. Generation-side guards can be bypassed by a new model, a cache, or an external API. The injection point is the only mandatory gate.

**Cost:** one extra check, effectively free.

---

## Decision 5: Borrow or give up?

**Situation.** A message refers to an image, but no local copy exists. The candidate pool contains an old image that is "close in time."

**Options:**
- Borrow the old image so there is something to show.
- Give up and mark "content not recognized."

**Choice:** give up. Use only candidates that no message has consumed. If none exist, return nothing.

**Why:** a wrong image is far worse than a missing one. When an image is missing, the user knows to resend. When the wrong image is attached, the system attributes A's image to B, with no error at all. That is the worst class of bug: **it looks correct and is wrong.**

A real case: two users' messages were 19 seconds apart. The pool reused an old file, the same image was decrypted twice, and it was attributed to A and then to B. The fix was "only unused candidates." Coverage dropped slightly; accuracy rose.

**Cost:** a few images that never landed locally will genuinely be missing. That is the intended trade-off.

---

## Decision 6: Block or degrade?

**Situation.** The caption service crashes or stops responding.

**Options:**
- Stall until it recovers.
- Degrade: push the text now, push the caption later.

**Choice:** degrade. At 90 seconds, push without media. If the caption finishes later, send it as a separate follow-up.

**Why:** chat is real-time. One stalled message queues everything behind it, and the user has no idea what you are waiting for. Better to say "I haven't read the image yet" than to freeze the conversation.

**Cost:** a two-part reply may occur — text first, caption later. That is better than blocking, but the follow-up must be a separate message and must not overwrite what was already sent.

---

## Extracting the Decisions into a Library

Four of the six decisions are pure policy: when to wait, how long, what to do when the caption is short, and what to do on failure. They do not depend on any platform and should not be scattered through business code.

So I extracted them into a small library: **multimodal-preprocess-kit**.

- https://github.com/AirXsLink/multimodal-preprocess-kit
- Pure functions. No network, no file I/O, no model calls.
- Five entry points: `decide`, `should_retry_caption`, `choose_caption_mode`, `can_use_thumbnail`, and `can_borrow_other_media`.
- Standard library only. Eight tests.

The core decision table:

| State | Action |
|---|---|
| Caption failed | `drop_media` (production usually maps this to a placeholder push) |
| Caption too short | `retry_caption` |
| Caption pending, within the window | `defer`, with remaining seconds |
| Caption pending, past the window | `push_without_media` |
| Caption ready | `push_now`; screenshots return `caption_mode="auto"` |
| No local file | thumbnail fallback (can be disabled) |
| Media from another message | not borrowed by default |

The README has a section called **"Where this kit deliberately stops."** It lists four gaps the library intentionally does not cover: failure semantics, garbage-caption detection, text-heavy retry gating, and thumbnail semantics. Those belong to the adapter, not to hidden library behavior.

---

## Afterword

None of the six decisions is "smarter." Each one picks an explainable default under uncertainty:

- Not sure you can read it → wait, but with a ceiling.
- Not sure the text was captured → retry once, but only for long images.
- Not sure the caption is valid → mark it invalid, but keep a placeholder.
- Not sure the image is theirs → give up; do not borrow.

Good multimodal fallback is not about model strength. It is about whether **every "I don't know" has a default answer.**

> If your bot has ever "not seen it, then seen it," do not swap the model first. Read its decision table.
