---
layout: default
---

# The Heartbeat Is Alive, the Messages Are Not: Silent Push-Channel Failures

> Ports are up, the connection is established, the heartbeat keeps arriving — three green checks, and not a single message came through. This is about separating "the channel is alive" from "the business is flowing," using real data as the baseline.
>
> A note on the numbers: the seconds, counts, and durations below are examples drawn from actual operation. They are not universal benchmarks.

---

**In short:** A healthy connection does not mean messages are flowing. Only business data proves that.

---

## 0. Mentions Go Unanswered While Everything Looks Fine

Three mentions in a group chat, and the bot answered none of them. This lasted 1 hour and 43 minutes.

The status check: nine ports up, the status page all green, the TCP connection between the bridge and the push service `ESTABLISHED`, a heartbeat every 25 seconds, not one missing.

**And no message came through.**

Nothing on this path reported an error. The push service had not crashed, the bridge had not disconnected, the network had not blipped. Every existing check passed, and from the user's side the bot had simply vanished.

---

## 1. All Three Green Checks Were Lying

Take the "normal" apart:

- **Port green**: the process is listening.
- **Connection green**: the socket still exists.
- **Heartbeat green**: the peer still sends `ping`.

All three prove "the channel is alive." None proves "business events are flowing." And a push service can break in exactly the third way: TCP connected, heartbeat running, messages not delivered.

---

## 2. What a Heartbeat Proves and Does Not Prove

It proves:

- the network path works;
- the peer process is alive;
- the connection was not cut by a middlebox.

It does not prove:

- business events are being produced;
- events are delivered to subscribers;
- subscribers receive new events rather than a replay.

The way to tell them apart was **a second subscription as a control**: open a fresh subscription of the same kind and watch for 30 seconds. It received only `: ping` lines, not a single `data:` event — while the message store clearly had new rows.

**Conclusion:** the bridge could read; the push side had stopped pushing for every subscriber. A heartbeat is the easiest health signal to fake, because it is too cheap to fail.

---

## 3. Reconcile Against Business Data

When the channel watermark is untrustworthy, use a **business fact** as the baseline. Here the most reliable fact was the newest inbound row in the message store.

Two steps:

**First, make the bridge report watermarks:**

- `line_age`: seconds since any line was received (heartbeats included);
- `push_age`: seconds since the last `data:` event;
- `push_last_create`: the largest timestamp among received messages;
- `started_ts`: process start time.

**Second, make the sentinel reconcile with the store:**

- push silence ≥ 180 seconds;
- and the store's newest inbound row is newer than `max(push_last_create, started_ts) + 60 seconds`;
- both true → the push channel is dead.

Why `started_ts` matters: without it the baseline is 0, so "no message received today" looks like "push stopped." Anchoring at process start makes the baseline "since this run."

---

## 4. After Detection: Restart, Cooldown, Cap

The sentinel restarts the push service. It came back in 5 seconds and the bridge reconnected on its own.

Self-healing needs three constraints:

- **15-minute cooldown**: a restart does not always fix it, and repeated restarts become a restart storm;
- **at most two restarts in a row**: leave a window for a human;
- **launch the same way the startup script does**: do not invent a new path. Starting the executable directly made it come up and exit immediately; the launcher from the startup script brought the port back in 5 seconds.

---

## 5. Do Not Kill "Nobody Is Talking"

If the rule were only "how long since the last message," it would fire every night when the group is quiet.

So the reconciliation also requires that "the store's newest inbound row is moving." At night the store is idle too, the check fails, and the sentinel does nothing.

This has a useful side effect: **"normal quiet" is now defined.** Quiet alone is not a failure; quiet while the store is updating is.

---

## 6. Why Fixing the Push Service Is Not Enough

Why the push service went silent can only be answered inside it — from the outside, all you get is "it stopped pushing."

So the outside can only do three things:

- **make the failure visible**: report watermarks, and show "push 12s ago / push silent 300s" instead of a flat "connected";
- **make the failure self-heal**: sentinel reconciliation plus restart;
- **bound the failure**: cooldown and a restart cap, so the worst case shrinks from "silent for hours" to "caught on the next sentinel tick."

If the root cause is out of reach, at least do not let it be silent.

---

## 7. Three Rules to Take Away

1. **Channel liveness is not business activity.** Ports, TCP, and heartbeats are liveness; only business events are activity. Monitor both, and never let one stand in for the other.
2. **Silence thresholds must be reconciled against a business baseline.** Time alone kills the genuinely quiet case; connection state alone misses the connected-but-not-pushing case.
3. **Self-healing needs a cooldown and a cap.** Otherwise one failure becomes a restart storm and takes observability down with it.

---

## 8. Afterword

The expensive lesson here is not "the push service can break." It is that we used three green checks to prove something they cannot prove.

> If your monitoring has an item called "connection healthy," ask it one question: when did it last prove that a message arrived?
