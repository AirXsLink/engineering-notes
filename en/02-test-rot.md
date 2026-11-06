---
layout: default
---

# How Tests Rot Quietly

> How a production test suite went from "all green" to "green and meaningless." Five cases, all from real regressions.
>
> A note on the numbers: all counts and ratios below are examples drawn from actual operation. They are not universal benchmarks.

**In short:** A test suite is not broken only when it turns red. The worst state is green while no longer testing anything.

**At a glance**

| Case | Failure |
|---|---|
| 1 | `object.__new__` bypassed `__init__`; production added attributes; the stubs drifted |
| 2 | The same trap was hit twice |
| 3 | 13/13 unit tests green, assembly broken, fail-open swallowed the import error |
| 4 | The window boundary was untested; production sent four replies |
| 5 | Constructed data skipped a real protocol field |

---

## 0. Three Red Tests

One day a regression run produced three errors: one `TypeError`, two `AttributeError`s.

The first assumption was "today's changes broke it." That was not the case. The three tests **had been broken for a long time**; nobody had run them recently.

Worse, the other seven tests were green. They looked like they were protecting the project, but each was only protecting the moment it was written.

The real danger of test rot is not a red test. It is a green test that no longer means anything.

---

## 1. Stub Drift: Tests Copy the Initialization Contract

Using `object.__new__` to build an object skips `__init__`. The test then patches in only the attributes it happens to need:

```python
bridge = object.__new__(Bridge)
bridge._user_id_names = {}
bridge._image_pending = {}
```

`__init__` promises "after construction, these attributes exist." Bypassing it copies that promise into the test. Every production change expires the copy.

### Case 1: Three tests, three attributes

Production later added:

- an attribute `_user_id_names`
- an attribute `_image_pending`
- an extra method parameter, `expected=""`

The stubs were not updated. Three tests failed with `AttributeError` and `TypeError` while **production code was completely healthy.**

The root cause is not "bad tests." It is an implicit contract between tests and production, duplicated by hand and never checked.

**Fix:** update only the tests — add the attributes and the default parameter. Production stays untouched.

**What should have happened:** either forbid `object.__new__` for production objects, or turn "which attributes the stub needs" into an explicit checklist that is enforced whenever an attribute is added.

### Case 2: The same trap, twice

A month later, two more instance attributes were added (a forward index and an ambiguity flag). Two stubs were missed again.

The first time was an oversight. The second time is a missing step in the process.

**Current rule:** before adding an instance attribute, search the codebase for `object.__new__`. Fix every hit.

---

## 2. Swallowed Exceptions: Green Unit Tests, Failed Assembly

### Case 3: The filter shipped and logged nothing

A content filter shipped with **13/13 unit tests green**. It was expected to block a fair amount of content.

In production, the logs showed zero `memory write skipped:` lines.

The first interpretation: "everything passed."

The real cause: the plugin directory lacked `__init__.py`. The import failed, the exception was swallowed, and the code fell back to fail-open. **The filter never loaded.**

**Why the unit tests missed it:** they imported the module directly and tested the function. Production used a plugin-loading path that required a package marker. **The tests covered logic, not assembly.**

The nastier part: "the filter allows everything" and "the filter never loaded" produce identical logs. Both are silent.

**Fix:** load by file path and log a warning when loading fails. Immediately after the fix: `memory write skipped: too_short(6)`.

**Takeaways:**
1. Green unit tests do not prove the integration works. Logic and assembly are different layers.
2. Every fail-open must log. Its purpose is to keep the main path running, not to hide the failure.

---

## 3. Coverage Blind Spots: Intervals Tested, Boundaries Missed

### Case 4: The five seconds after the window expired

The rule: a 90-second window starting at the first message; at most one reply per window; a new draw after expiry.

The tests covered two cases: the second message inside the window is blocked, and the first message outside the window passes. **All green.**

Production sent four replies to one topic.

The third case was missing. A message could be **received** inside the window but only **claim the slot** after a 5-second buffer. By then the window had expired. The code saw the expiry, created a new window, reset the slot, and let one more reply through.

"Inside" and "outside" were both tested. Nobody tested "the window just expired while the buffer was still running."

**Fix:** an expired window returns "no slot"; the next message starts a new draw. Add 15 boundary tests, three of which target "the instant of expiry."

**Takeaway:** state-machine bugs live at boundaries. Test "just expired," "exactly equal," and "just past the edge," not two calm intervals.

### Case 5: Replay real data

Quote replies: when a user quotes a message, the bot must know what is being answered.

Unit tests built from constructed data passed. In production, a user quoted a message and replied with "。"; the bot answered something unrelated.

The quoted content lives in a nested quote block. The parser took only the new sentence; the quote block was dropped. Constructed data never contained that field, so the tests never covered it.

**Fix:** parse the quote block into a `[Quote from X: original text]` prefix, and add 12 tests, two of which replay the raw fields of real messages.

**Takeaway:** keep at least one real-data replay. Constructed data quietly avoids the fields you did not think of.

---

## 4. Five Rules

1. **Do not build production objects with `object.__new__`.** Use a factory or the constructor. If you must bypass it, write the attribute checklist next to the production class.
2. **Search for `object.__new__` before adding an instance attribute.** One grep, then fix every hit.
3. **Every fail-open must log a warning.** A silent fallback is not installed. When two states look identical, the log is the only differentiator.
4. **Test state-machine boundaries, not intervals.** "Exactly equal," "just expired," "just past the edge" are worth more than "inside" and "outside."
5. **Keep at least one real-data replay.** Constructed data proves the path you thought of; real data proves the fields you did not.

---

## 5. Afterword

Tests are not a one-time asset. They share an implicit contract with production: what the constructor promises, which fields exist, where the boundaries are. When the contract changes, tests either turn red or pretend to be green.

**Red is good — at least you know. The dangerous state is green, and nobody remembers what it tests.**

> If your regression suite uses `object.__new__`, grep it today. It will not raise an error. It will simply tell you, one day, that it has been broken all along.
