---
layout: default
---

# Making "It Doesn't Look Like the Character" Measurable

> The hardest part of training a character LoRA (Low-Rank Adaptation) is not training it. It is knowing whether the result looks right. This article turns a subjective complaint into three reproducible metrics.
>
> A note on the numbers: the ratios, weights, sample counts, and costs below are examples drawn from actual operation. They are not universal benchmarks.

**In short:** Split "not similar" into three numbers: head-to-body ratio, same-seed A/B, and blind hit rate.

---

## 0. "Not Similar" Is Not Actionable

The first character images came back, and the feedback was: "It doesn't look like her."

So I started tuning. Prompts, weights, samplers. Each time I asked, "Does it look right now?" The answer was "still not really."

The problem was not the model. The problem was that **the feedback had no executable shape.** "Not similar" could mean the proportions are off, the hair color is off, the composition is off, or the face looks too mature. Each has a different fix, but one sentence covered all of them.

Then I split "similar" into three metrics, each expressible as a number. Tuning stopped being guesswork and became reading a table.

---

## 1. Metric 1: Head-to-Body Ratio

The character is chibi, so proportion is the easiest thing to measure and one of the easiest for a model to distort.

The method: measure shoulder width divided by head height in the output.

| Source | Ratio |
|---|---|
| Reference illustration | 1.48 / 1.51 |
| Standing pose (anchored) | 1.52 / 1.50 |
| Action pose (text-to-image) | 1.18 / 1.25 |
| Action pose (anchored to a sitting frame) | 1.41 / 1.90 |

The gap is obvious: the standing pose matches the reference almost exactly; the action pose is 0.3 off. On screen, 0.3 means "longer face, sharper chin, looks like an adult."

Once the number exists, the fix is obvious: the action pose cannot rely on prompts alone; it needs reference-image anchoring. After anchoring, the ratio moved from 1.18 to 1.41, close to the reference.

**Watch the 1.90 next to 1.41.** That is a sitting pose opening the hips, inflating the shoulder measurement. It does not mean the proportion improved. Metrics must be read with the pose in mind.

---

## 2. Metric 2: Same-Seed A/B

The eye is good at comparing two images that differ in exactly one variable. It is almost useless at comparing two images with different seeds.

So change one variable at a time, fix the seed, and produce a comparison grid:

| Variable | Change | Result |
|---|---|---|
| Prompt | narrow face / sharp chin → wide face / big eyes | Closer to reference; right direction |
| IP-Adapter weight | 0.6 → 0.8 | **Largest single gain**: rounder face, bigger eyes |
| LoRA weight | 0.7 → 0.9 | Worse: narrower face, reddish-brown eyeliner |
| Color matching | on → off | Cannot be disabled: hair turns sky blue, image washes out |

Two of the four results are counterintuitive:

- **Stronger is not better.** LoRA at 0.9 looked less like the reference than at 0.7. "More character constraint" and "closer to the reference" are not the same thing.
- **Color matching is not optional.** It looks like a finishing touch, but turning it off produces visible color drift.

Without same-seed A/B, both traps would have been missed.

---

## 3. Metric 3: Blind Hit Rate

The first two metrics measure "similar to the reference." This one measures "can a stranger tell it is the same character."

Method: produce 10 groups of images, mix in other characters, and ask someone to blind-pick which images belong to the same character.

The acceptance line: **at least 7 of 10 groups are identified.**

This simulates the real scenario. People in a group chat will not compare pixels against a reference sheet; they will glance and decide "that's her." Blind recognition is the real bar.

---

## 4. A Counterintuitive Finding: Prompts Have a Ceiling

The cyan tips on the hair are a distinctive feature of the reference. I tried prompt weighting, up to 1.35. Nothing.

The fix came from the **data side**: crop the hair tips into separate training samples, one caption per image, retrain. The cyan tips appeared.

The harder conclusion: **prompts can only adjust what the model already knows how to draw. If the model has never seen it, no weight will produce it.**

---

## 5. Dataset Statistics Beat Parameter Tweaking

The action pose kept failing at "sitting and holding a bowl of white rice." Many prompt revisions, no improvement.

Then I counted the training set: of 184 frames, **177 had props = none.** The rest were 3 cookies, 1 loaf of bread, 1 donut, and 1 lollipop.

The model had no concept of "white rice." No prompt could fix that.

The fix was not another prompt revision. It was more data: 8–10 sitting poses, 6–8 holding a bowl or eating, 3–5 hand close-ups, then retrain.

**Count first, tune second.** If it is not in the dataset, tuning is gambling.

---

## 6. When Hardware Is Short, Do the Memory Math

The first plan for this pipeline was IP-Adapter. The math: SDXL 6.94 GB + CLIP Vision 2.5 GB ≈ 9.4 GB, against an 8 GB card. Dead on arrival; no need to try.

The second plan was local LoRA training, but 8 GB of VRAM plus the then-current RAM price made it a bad deal.

The choice was cloud training: 433 image pairs, 2600 steps, loss 0.0509, about 2.7 yuan per run, then download the LoRA for local inference.

**Do the memory math before choosing a route.** Many "quality problems" are actually "the hardware cannot run this route at all," and no amount of tweaking will help.

---

## 7. A Reusable Process

```
1. Define metrics: head-to-body ratio, same-seed A/B, blind hit rate
2. Measure the baseline: reference and current output; the gap is the target
3. Change one variable: prompt, weight, reference image, or dataset
4. Measure again: if the number did not move, nothing changed
5. Decide the ceiling: two rounds without improvement → move to the data side
```

Step 4 is the important one: **if the number did not move, nothing changed.** That sentence eliminated half of my wasted tuning.

---

## 8. Afterword

Character consistency is not "drawing more accurately." It is "reproducing the same face every time."

Models are probabilistic. You cannot guarantee that every output is right. You can guarantee that **every output can be measured, and you can explain why it is wrong.**

> If you are training a character LoRA: before tuning anything, measure the head-to-body ratio of the reference.
