---
layout: default
---

# Quoting Someone Else's Image: Who Claims the Media?

> A user quoted a snow photo another group member had just posted and asked the bot, "Can you see this?" The bot answered: "No — the quote contains only the three characters [image]." Three layers stacked up to turn a real photo into three characters.
>
> A note on the numbers: the times, pixel sizes, character counts, and seconds below are examples drawn from actual operation. They are not universal benchmarks.

---

**In short:** Anti-mixup rules will hurt explicit quotes. And the first sentence sent is final.

---

## 0. A Snow Scene Reduced to Three Characters

Someone posted a snow photo in a group. Another member quoted it and mentioned the bot: "Can you see that someone sent a snow scene?"

The bot replied:

> I can't see it. The quote only contains "[image]" — the actual picture didn't come through.

The photo was real, posted a minute earlier. But the bot genuinely had only three characters — this was not the model hallucinating; the context really contained nothing else.

---

## 1. The Quote Block Holds Only a Placeholder

A reply-with-quote is a nested structure in the protocol: the quoted content is attached to the new message as a quote block.

Text quotes carry the original words; image quotes carry only a placeholder, because the quote block does not embed the image itself. From the bridge's point of view, the message is:

```
[quote: some member: [image]]
Can you see that someone sent a snow scene
```

That is exactly what the model saw. It had no image and could not describe one.

---

## 2. The Claiming Rule: Only Your Own Pending Media

Incoming media enters a pending store and is "claimed" — either when someone references it, or on timeout — before it is described.

Claiming had one anti-mixup rule: **only claim media from the same sender.** It was added after "A's image was attributed to B," and it worked.

But it also hurt explicit quotes: the user was quoting **someone else's** image, the sender did not match, and the claim failed.

**The anti-mixup rule was right. What it lacked was an exit for explicit quotes.**

---

## 3. Fix One: Extract the Quoted Author

The quote block has no sender ID, but it does have a name:

```
[quote: some member: [image]]
```

So the quoted author can be extracted from that shape, and claiming can cross senders by name.

Two constraints:

- **Cross-sender only on an explicit quote**: without the parameter, behavior is exactly as before;
- **No match, no media**: the quote block has no ID, so the comparison is by name. If the name does not match, give up — better to miss the image than borrow the wrong one.

---

## 4. Second Layer: A Photo Treated as a Screenshot

Even with the claim fixed, a second gate remained: the 1276×1702 snow **photo** was classified as a text-heavy image by the "long edge ≥ 700" rule and sent through line-by-line transcription.

The transcription returned 15 characters: "This is a photo of trees covered in snow." Correct, but too short — which triggered a "too short, retry" path that **retried transcription**, wasting another 40 seconds.

**A photo is not a screenshot.** The long-edge rule can only say "this might be a text image," not "this is one." When the result is too short, the right move is a different question — "just describe the picture" — not the same question again.

---

## 5. Third Layer: The First Sentence Is Final

The description finally arrived at 16:01:48. But a reply had already gone out at 16:00:20 ("I can't see it"), so the "this discussion has already been answered" gate turned the late description into "context only" — the image entered the context, but **the user would never get a correction.**

The first sentence sent is final.

This layer was not fixed in the bridge: one reply per discussion is a design choice, not a bug. Two things can be done:

- make the description correct sooner (the previous layer's fix);
- accept that late-correct information may not be corrected, so the first reply should be more careful about uncertainty — ask instead of asserting "I can't see it."

---

## 6. Only the Three Together Say "I Can't See It"

| Layer | Problem | Fix |
|---|---|---|
| Claim | only claims the sender's own pending media | explicit quotes claim across senders by name |
| Description | photo classified as text image; short retry repeated transcription | a short result asks "just describe the picture" |
| Fallback | already answered means no correction | design tradeoff; the first two layers reduce how often it fires |

Each layer has a reason on its own: anti-mixup, transcribe long images, one reply per discussion. Stacked together, the user gets "I can't see it."

---

## 7. Three Rules to Take Away

1. **Leave an exit for explicit quotes.** A quote is evidence supplied by the user, stronger than a heuristic; a rule may cross senders, but only when the target is explicit.
2. **"Short" does not always mean "wrong type," but do not retry the same question.** A different question is more likely to produce usable information than a repeated one.
3. **Answer where late-correct information goes.** If the first sentence is final, its wording about uncertainty must be more conservative.

---

## 8. Afterword

The sharpest part was not any single rule, but the sentence "the actual picture didn't come through" — it turned "I failed to claim it" into "it was never sent."

> If your system has ever said "I didn't receive it," check first: did it not receive it, or did it fail to claim it?
