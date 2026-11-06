---
layout: default
---

# 14 Failures That Never Threw an Error

> A field guide to silent failures in a self-hosted chatbot. Every case comes from a production deployment. Timestamps, counts, and log lines are real; identities, platforms, and paths have been redacted.
>
> A note on the numbers: all timestamps, counts, and ratios below are examples drawn from actual operation. They are not universal benchmarks.

**In short:** None of these 14 failures threw an exception. Every one looked normal. The most expensive failures are the ones that never report themselves.

**At a glance**

| # | Failure |
|---|---|
| F1 | The whole stack was down for 25 hours before anyone noticed |
| F2 | It got slower before it crashed: an 8 GB card running a 16k context |
| F3 | False reds in smoke tests: services were not ready yet |
| F4 | A tolerance threshold blocked the shortest name |
| F5 | "Version unknown" during a restart |
| F6 | fail-open swallowed the exception; the filter never loaded |
| F7 | Three tests had been broken for a while; nobody ran them |
| F8 | 61 "empty messages" were actually multi-line log entries |
| F9 | A single "□" made the model believe the image never arrived |
| F10 | The same image was attributed to two people within 19 seconds |
| F11 | "Can't read it" was reported as "didn't receive it" |
| F12 | One nickname change split a user's memory in two |
| F13 | One discussion received four replies |
| F14 | Quote replies showed only the new sentence |

---

## 0. Twenty-Five Hours

One afternoon someone asked: "Is the bot gone?"

It had been down for 25 hours. Seven of nine ports were down, and no process was running. The logs showed no crash. The last line was "connection lost, reconnecting in 5 seconds," and after that, silence.

The process had not crashed. It had been stopped, and nothing restarted it. The machine had been up for six days, but the startup entries covered only the memory service. The message bridge, the dialogue service, and the image generator all depended on someone starting them by hand.

That day set the priority for every later investigation: **the most expensive failure is not the one that reports itself. It is the one that looks fine.**

---

## 1. Down, Without a Crash

Two cases, one theme: a service can stop working without ever telling you.

### F1 · The stack was down for 25 hours

**Symptom.** The status page refused connections. Seven of nine ports were down. No process existed. The logs ended 25 hours earlier.

**Cause.** No supervision. Once the processes were stopped or killed, nothing brought them back. Startup entries covered only the memory stack; the core services were started by hand.

**Fix.** A port sentinel that probes all nine ports every five minutes. If something is missing, it waits 60 seconds and checks again; only then does it start the stack. The important part is telling "it crashed" apart from "you stopped it yourself": the shutdown script writes a pause marker, and the sentinel does nothing while that marker exists.

**Verified.** A real failure drill: kill the bridge process, and the stack recovers in nine seconds.

**Takeaway.** Every rule needs a check that can fail. A rule without a check is just a wish.

### F2 · It got slower before it crashed

**Symptom.** Image captioning returned HTTP 500, and the underlying process exited with `0xc0000409` (stack overflow). Before that, one image had been captioning for 240 seconds.

**Cause.** An 8 GB GPU, a 6.6 GB vision model, and a 16384-token context. It did not fit. `ollama ps` showed `19%/81% CPU/GPU`: 19% of the model had spilled to the CPU, and then the process died. The desktop baseline was using about 3.9 GB of GPU memory that day; the same image took 29 seconds when the baseline was only 1.4 GB.

**Measured** (same image, same prompt):

| Context | Time | Output | State |
|---|---|---|---|
| 16384 | 66.1s | 283 chars | 19% spilled |
| 8192 | 19.6s | 280 chars | 14% spilled |
| 4096 | — | HTTP 400 | hard failure |

**Fix.** Default to 8192. Allow 16384 only as a retry when at least 6000 MB is free. Treat 4096 not as "the lowest setting" but as unusable.

**Takeaway.** A default is not a safe value. Before a crash, the system usually gets slower first. **Slowness is a warning, not noise.**

---

## 2. Swallowed Errors

When fallback code eats exceptions, the system goes quiet in exactly the way a healthy system does.

### F3 · False reds are more expensive than real reds

**Symptom.** The full smoke suite reported 6/12, then 10/12. It looked like a broad regression.

**Cause.** The first run hit a cold start; services were not ready. The second run counted a cold-start traceback from the image generator as a "real exception since startup."

**Fix.** Wait for readiness before running: all nine ports must answer, and the image queue must respond. Cap the wait at 90 seconds; on timeout, log a warning and run anyway. A real failure should still fail.

**Why not whitelist the traceback.** That would also wash away a genuine "image generator is down." Fixing the cause (don't test before readiness) is safer than relaxing the check.

**Takeaway.** False reds train people to ignore alerts. An alert that loses credibility is no alert at all.

### F4 · A tolerance threshold blocked the shortest name

**Symptom.** A message to the owner was blocked by the pre-send check. Three retries failed, and the bot paused replies to that contact for 15 minutes.

Log: `title mismatch OWNER: best_ratio=0.750 best='Ofu,%轩'`.

**Cause.** The loose-matching path required the expected name to contain at least four CJK (Chinese, Japanese, Korean) characters. The owner's display name had only two. The tolerance logic **never ran**; the check fell through to strict matching, and 0.75 was below 0.85. Optical character recognition (OCR) had also read one character as `%`.

**Fix.** A title whitelist that admits by fragments (both `FU` and `轩` must appear), applied only to registered names. Add a negative test: a similar-looking name must not pass.

**Takeaway.** Thresholds hurt the shortest, rarest, and most deserving names first. **When you add a threshold to a fallback, ask who falls below it.**

### F5 · "Version unknown" right after a restart

**Symptom.** The smoke suite occasionally reported the persona version as `vNone`.

**Cause.** The dialogue service had just restarted and had not written the version line yet. A rerun passed. It was not a regression.

**Takeaway.** Some "failures" are startup ordering, not logic errors. Documenting them as known jitter is cheaper than repeatedly fixing a problem that does not exist.

### F6 · The filter shipped, and not one log line appeared

**Symptom.** A memory-write filter went live and was expected to block a fair amount. Yet there was not a single `memory write skipped:` line. It looked like "everything passed."

**Real cause.** The plugin directory lacked `__init__.py`. The import failed, and `except` swallowed it into fail-open (allow). **"The filter allows everything" and "the filter never loaded" look identical in the logs.**

**Fix.** Load by file path, and log one warning when loading fails. Do not stay silent.

**Verified.** Immediately after the fix: `memory write skipped: too_short(6)`, which also proved the code was reading the user's original words rather than the bot's reply.

**Takeaway.** Every fail-open must leave a log line. The point of fail-open is "don't block the main path," not "say nothing."

### F7 · Three tests had been broken for a while

**Symptom.** A regression run produced one `TypeError` and two `AttributeError`s. It looked like that day's changes had broken something.

**Real cause.** Unrelated to that day's work. The tests built objects with `object.__new__`, bypassing `__init__`, and patched in a few attributes by hand. Production had since added three instance attributes and one method parameter. The stubs were never updated, so the tests **had been broken for a long time — nobody ran them.**

**Fix.** Change only the tests; zero production changes. Add the missing attributes and default parameters.

**Takeaway.** A test you do not run is not a test. It is a false sense of coverage. `object.__new__` is hidden coupling: search for it before adding any instance attribute.

---

## 3. Wrong but Plausible

The status is right; the meaning is wrong. Nothing stops.

### F8 · The 61 "empty messages" were not empty

**Symptom.** The logs showed 61 instances of "skipping blank text segment (1 char)," which looked like 61 empty messages. At the same time, 61 of 97 replies began with a blank line.

**Cause.** Model output started with a newline, and the outbound sanitizer handled Markdown but not leading/trailing whitespace. In the log, the first line of the body was blank, which looked like an empty message. It was a multi-line log artifact.

**Fix.** Add `strip()` at the end of the outbound sanitizer.

**Takeaway.** Log shape can mislead. Before concluding "61 empty messages," check whether you are looking at 61 multi-line entries.

### F9 · One "□" made the model believe the image never arrived

**Symptom.** A user sent an image, and the bot replied: "This image is empty. I can't see anything." It had received the image; the caption was simply useless.

**Cause.** The transcription prompt said "use □ for characters you cannot read." The model complied and returned a single `□`. That `□` entered the context as a valid caption, while another instruction said "if it is marked as content-not-received, tell the user to resend." The model conflated a bad caption with a missing image.

**Fix.** Add a garbage check: after removing punctuation, fewer than three meaningful characters means "content not recognized." Apply the check **at the injection point**, not only at the generation point. Generation-side guards can be bypassed by a new model, a cache, or an external API; the injection point is the last gate.

**Takeaway.** A failed caption and a missing image are two different states. They must not share one sentence.

### F10 · The same image was attributed to two people within 19 seconds

**Symptom.** The logs showed the same file decrypted twice: the image was first attributed to A, then to B 19 seconds later.

**Cause.** The candidate pool mixed fresh and stale files. Once fresh candidates ran out, the code fell back to reusing old ones. Thumbnails are re-encoded, so their MD5 never matches; they were guarded only by a 30-second time window. The two messages were 19 seconds apart, both inside that window.

**Fix.** Use only candidates that have not been used by any message. If none exist, give up. Better to miss this image than to borrow someone else's.

**Cost.** Slightly lower coverage, higher accuracy. It matches the principle already in the code: **a fallback strategy must include "give up," or the fallback manufactures errors.**

### F11 · "Can't read it" was reported as "didn't receive it"

**Symptom.** After 18:56, several images could not be read. People in the group started asking, "What's wrong with her?"

**Cause.** Two layers. First, the missing garbage check from F9. Second, the prompt distinguished only "content not received," not "content not recognized." The image had arrived but was unreadable, yet the model still said "I didn't get it, send it again."

**Fix.** Split the two states. For "not received," ask the user to resend. For "not recognized," explicitly say do not claim it was not received, and ask the user to describe it in text.

**Takeaway.** To the user, "not received" and "can't read it" are both "you didn't see my image." To the system, they are two states that must be modeled separately.

### F12 · One nickname change split the memory

**Symptom.** Media messages often carried no sender ID, and the lookup failed, so memory was keyed by nickname. When a user changed their nickname, their history split into two buckets.

**Fix.** Build a reverse index from nickname to ID, learning only from text messages (the sender field on media messages is unreliable). If one nickname maps to two IDs, mark it ambiguous and refuse to use it. Falling back to the nickname is better than misidentifying a person.

**Related lesson.** This change added two instance attributes and hit the F7 trap again. Making the same mistake twice means it is not carelessness; it is a missing step in the process.

---

## 4. At the Edges

Interval tests pass. Bugs live at the boundary.

### F13 · One discussion received four replies

**Symptom.** Between 17:16 and 17:18, one topic received four replies. Three of them discussed the same thing. To the user, "she is glitching."

**Cause.** Three seams overlapped:

1. Mentions always get a reply. That was by design.
2. Emoji pushes did not consume the "one reply per discussion" slot, but the payload still contained the entire discussion. The model answered the same link 40 seconds after it had already answered it.
3. **The window quietly recreated itself after expiry.** The window started at the first message and lasted 90 seconds. A message could be received inside the window but only claim the slot after a 5-second buffer, when the window had expired. The code saw the expiry, created a fresh window, reset the slot, and let one more reply through.

**Fix.** An expired window returns "no slot," never a new window. The next message starts a new draw. If the discussion has already been answered, images and emoji only enter the context and no longer trigger a reply.

**Verified.** 15 boundary tests passed: the second message in the same window is blocked; expiry does not recreate the window; expiry starts a new draw; emoji does not trigger a reply in an answered discussion; a mention overrides the draw.

**Takeaway.** The most fragile point of a state machine is **the instant of the boundary**. Test "the window just expired," not just inside and outside.

### F14 · Quote replies saw only the new sentence

**Symptom.** A user quoted a message and replied with just "。". The bot answered something unrelated.

**Cause.** The quoted content lives in a different field (a nested quote block). The bridge took only the sentence the user typed and dropped the quote entirely. The model had no idea what was being answered. The protocol did carry `localType=244813135921` and `<refermsg>`; the code simply did not parse them.

**Fix.** Parse the quote block into a `[Quote from X: original text]` prefix. An @ inside the quote does not count as calling the bot. Deduplication compares only the new sentence.

**Verified.** 12 tests, including two replays of real messages, image quotes, no-quote cases, @ inside the quote, and empty quotes.

**Takeaway.** A missing protocol field is usually not "no data"; it is "no parser." When the model answers the wrong thing, suspect a missing piece of context before blaming the model.

---

## 5. Three Rules to Take Away

1. **Every fail-open must log.** A silent fallback is not installed. When two states look identical, the log is the only thing that can tell "all passed" from "never loaded."

2. **Every rule needs a check that can fail.** Port probes, version fingerprints, readiness waits, pause markers — a rule without a check is a wish.

3. **A test you do not run is not a test, and it still creates false coverage.** Stubs are hidden coupling. Every instance attribute added to production is a chance for a silent drift.

---

## 6. Afterword

Only some of these 14 cases came from "wrong code." Most came from **state, timing, fallback, and protocol parsing**. Their common shape: the process is alive, the logs are normal, and the user can feel that something is off.

I later extracted three of the multimodal decisions — short-caption retry, deferral while the caption is pending, and failure fallback — into a small standalone library. The rest stay in this archive as an index for the next investigation.

> If you also run a self-hosted service that "looks fine": add a check that can fail, then add a fallback that does not swallow its own exceptions.
