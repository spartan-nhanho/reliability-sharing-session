# Presenter study guide

Background reading for whoever presents the deck. The slides show the short version. This guide gives you the long version, so you can explain each idea in your own words and answer follow-up questions without guessing.

Read it once end to end, then reread the section for each block the day before. The presentation script (`02-presentation-script.md`) tells you *what to say and when*. This guide covers *why it's true*.

Slide numbers in brackets, such as [slide 10], refer to the deck.

---

## 0. The whole session in four sentences

The closing slide [slide 29] is the summary. If an intern remembers only this, the session worked:

1. **If it matters, it's a row.** Work that must happen after a request is written to the database in the same transaction as the business data. A worker makes it happen later.
2. **Dedupe, commit, ack.** Queues deliver at least once, so every consumer must handle the same message twice. Acknowledge only after the work is committed.
3. **Fail fast and alone.** A slow dependency must not take the whole service down. Timeouts, limited retries, circuit breakers and separate pools keep the damage local.
4. **Page on symptoms.** Alert on what users feel (errors, latency, work piling up), not on causes like CPU. Every async hop needs its own "is work piling up here?" signal.

The opening line [slide 2] explains why the session exists: *most outages aren't bugs in the code; they're the code meeting a slow dependency it never planned for.*

---

## 1. The dual-write problem [slides 4–5]

### What it is

A request often needs to do two things in two different systems, for example:

- save an order in **Postgres**, and
- publish an `OrderCreated` event to **SQS** (or send an email).

Postgres and SQS do not share a transaction. You can't say "commit both or neither". So there is always a moment between the two writes where a crash leaves them out of sync.

```kotlin
transaction(db) {
  orders.insert(order)
}
sqs.send(OrderCreated(order.id))   // ← the pod can die here
email.send(confirmation)
```

### Why a pod dies at exactly that moment

It sounds unlikely, but it happens every day at scale:

- **Deploys.** Every rolling deploy kills every old pod. If you deploy 10 times a day with 10 pods, that's 100 kills a day.
- **OOM kills.** The container passes its memory limit and the kernel kills it immediately. No cleanup code runs.
- **Node drains.** Kubernetes moves pods off a node for upgrades or autoscaling.
- **Network blips.** The `sqs.send` call can also simply fail or time out.

At low traffic you never see it. At real traffic it becomes "a few customers a week never got their email", and nobody can explain why.

### The four outcomes [slide 5]

| | Send succeeds | Send fails or never runs |
| --- | --- | --- |
| **Commit succeeds** | Happy path | **Lost event**: order exists, no event, no email |
| **Commit fails** | **Ghost event**: email for an order that doesn't exist | Nothing happened; client sees an error and retries |

Two of the four outcomes are honest. Two are bugs.

**Swapping the order doesn't fix it.** Send first, then commit, and you trade the lost event for the ghost event. Putting the send *inside* the transaction block has the same problem: the send goes out, then the commit fails.

### Why not a distributed transaction (2PC / XA)?

Interns who've read about databases may ask this. Short answer:

- SQS, email vendors and most HTTP APIs don't support two-phase commit at all.
- Where 2PC exists, it's slow, it holds locks across the network, and a coordinator failure can leave both sides stuck.

So we make **one system the source of truth: the database**. That's the transactional outbox.

### The three rules [slide 4]

What a request is allowed to do:

1. **Write to the database.** One transaction, every table it touches, *including a row that says "something else must happen"*.
2. **Respond.** The caller gets `201` when the commit lands, not when the email went out.
3. **Nothing else.** No fire-and-forget coroutine (`launch { }`), no "best effort" send after commit.

> "If it matters, it's a row; a worker makes it happen."

Why `launch { }` is on the list: a coroutine (or thread, or `CompletableFuture`) started from a request handler lives only in memory. If the pod dies, the work disappears and nothing records that it was ever planned.

---

## 2. Idempotency keys [slides 6–7]

### Two different kinds of duplicate

This is the key point of the slide. There are two different sources of duplicates, and each one needs its own fix:

| Duplicate | Cause | Fix |
| --- | --- | --- |
| **Same message delivered twice** | SQS redelivers (at-least-once) | Consumer dedupes by **event id** |
| **Same user action sent twice** | Client timed out and retried | API dedupes by **idempotency key** |

The trap: the mobile app times out at 10 s and retries, but the first request *did* commit. Now the API creates a **second order** with a **second, different event id**. The consumer's dedupe sees two different ids and processes both. The customer is charged twice.

The outbox alone can't catch this. It faithfully records whatever the API tells it.

### How it works

The client generates a key (a UUID) **once per user intent** ("the user tapped Pay"), and sends the same key on every retry of that intent.

```kotlin
// POST /orders
// Idempotency-Key: 9f1c2a…
transaction(db) {
  idempotency.find(orgId, key)?.let {
    return it.storedResponse          // replay
  }
  val order = orders.insert(…)
  outbox.insert("OrderCreated", …)
  idempotency.insert(orgId, key, 201, order)
}
```

- The key, the order and the outbox row are written in **one transaction**. All three land, or none.
- A **unique constraint on `(org_id, key)`** handles the race. Two duplicates arrive at the same time, both find nothing, both try to insert. Postgres makes the second insert wait on the unique index until the first transaction commits, then fails it with a unique violation. Your handler catches that error, reads the stored row and returns the stored response.
- Scoping by `org_id` means one tenant can't collide with or probe another tenant's keys.

### Details worth knowing

- **New key per attempt defeats the point.** If the client makes a new UUID on every retry, every retry looks like a new intent.
- **Same key, different body.** A good implementation stores a hash of the request body. If a retry reuses a key with a different body, return an error (`422` or `409`), not the old response. That's a client bug and it should be loud.
- **What to store.** Store just enough to rebuild the response (status + order id), especially if the full response holds sensitive data.
- **How long to keep keys.** Longer than any client's retry window. 24 hours is common (Stripe uses 24 hours). A scheduled job deletes old keys, the same way it cleans the outbox.
- **Standards.** There's an IETF draft for the `Idempotency-Key` HTTP header, and Stripe's API is the well-known example. Good reading for curious interns.

### The sequence [slide 7]

1. First request commits: key + order 1042 + outbox row.
2. The `201` response is lost (the client gave up at 10 s).
3. The client retries with the **same key**. The API finds it.
4. The API replays the stored `201 · order 1042`. One intent → one order, one event, one charge.

---

## 3. The pipeline map [slide 8]

This slide is the map for the rest of the session. Make sure everyone understands it before moving on.

```
Client ──► API ──(one transaction)──► Postgres: idempotency_keys, orders, outbox
                                          │
                                        Relay (on every pod, polls outbox)
                                          │
                                          ▼
                                     SQS orders-events ──► Consumer ──► processed table
                                          │                    │
                                 after 5 receives              └──► Email vendor
                                          ▼
                                     orders-dlq (alarm if not empty)
```

Four stages:

1. **Request.** Key, order and outbox row in one transaction. Then `201`.
2. **Relay.** Claim a batch, commit, publish to SQS, mark sent.
3. **Consumer.** Dedupe in `processed`, do the work, commit, ack. Then the email.
4. **Poison message.** A message that fails 5 receives moves to the DLQ, and the alarm pages.

The phrase on the slide: *one durable row in the request, then at-least-once all the way down.* Each hop may repeat work. None of them may lose it. Dedupe turns repeats into no-ops.

---

## 4. The transactional outbox [slide 9]

### The idea

Instead of sending the event directly, write it as a row in an `outbox` table **inside the same transaction** as the business data. A separate loop (the relay) reads unsent rows and publishes them.

Now there's only one commit, so there's only one outcome: both rows exist, or neither does. Publishing can be retried as many times as needed, because the row stays there until it's marked sent.

### A typical table

```sql
CREATE TABLE outbox (
  id             bigserial PRIMARY KEY,
  event_id       uuid        NOT NULL UNIQUE,   -- the id consumers dedupe on
  type           text        NOT NULL,          -- "OrderCreated"
  version        int         NOT NULL DEFAULT 1,
  org_id         uuid        NOT NULL,
  payload        jsonb       NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now(),
  claimed_until  timestamptz,
  sent_at        timestamptz
);

-- small, fast index over only the rows the relay cares about
CREATE INDEX outbox_unsent ON outbox (id) WHERE sent_at IS NULL;
```

The deck's slides don't show this schema. It's a reasonable example; check the real one in your repo before the session.

### Three parts, three owners of correctness

| Part | Its job | Guarantee |
| --- | --- | --- |
| Request | Make it **durable** | The event exists if and only if the order exists |
| Relay | Make it **delivered** | Every row is published at least once |
| Consumer | Make it **happen once** | Repeats are detected and skipped |

### Cleanup

Delete or archive sent rows on a schedule (for example, sent more than 7 days ago). If you don't, the outbox becomes the biggest table in the database and every query on it slows down.

---

## 5. The relay [slides 10–11]

This is where the subtle bug lives, so spend time here.

### The SQL, line by line

```sql
-- 1 · claim (its own short transaction)
UPDATE outbox
   SET claimed_until = now() + interval '30 seconds'
 WHERE id IN (
   SELECT id FROM outbox
    WHERE sent_at IS NULL
      AND (claimed_until IS NULL OR claimed_until < now())
    ORDER BY id LIMIT 100
    FOR UPDATE SKIP LOCKED)
RETURNING id, type, payload;
-- commit

-- 2 · publish each to SQS, no locks held
-- 3 · mark what was actually sent
UPDATE outbox SET sent_at = now()
 WHERE id = ANY(:published_ids);
```

- `sent_at IS NULL`: only unsent rows.
- `claimed_until IS NULL OR claimed_until < now()`: not claimed, or the claim has expired (the pod that claimed it probably crashed).
- `FOR UPDATE SKIP LOCKED`: lock the rows we pick, and *skip* rows another pod has locked at this instant instead of waiting for them. This lets 10 pods run the relay at once without blocking each other. (Postgres 9.5+.)
- `LIMIT 100`: batch size.
- `SET claimed_until = now() + 30 s`: a **lease**. "I'm working on these for the next 30 seconds."
- **Commit.** The row locks are released here. The claim column now does the job the lock used to do.
- Publish to SQS. No locks held. If SQS is slow, nothing in the database waits.
- Mark only the ids that were actually published.

### Why you never hold a row lock across a network call

The "obvious" relay is one transaction: `SELECT … FOR UPDATE`, publish, `UPDATE sent_at`, commit. On a normal day the publish takes 20 ms and nobody notices. On the day SQS takes 1 s [slide 11]:

- **Every relay pod holds a database connection for the whole second.** Connections are a small, shared pool. The API starts waiting for connections.
- **Row locks are held for the whole second.** The cleanup job and anything else touching those rows wait.
- **Long transactions hurt Postgres generally.** They keep old row versions alive (vacuum can't clean them) and make lock queues longer.

Slide 11's numbers (illustrative): locks held for **1.08 s** before, versus **40 ms + 15 ms** after.

The rule generalizes: **no network calls inside a database transaction.** It applies to the consumer too [slide 14].

### What happens on a crash

- **Crash after the claim, before publish.** The rows stay claimed until `claimed_until` passes (30 s). Then any pod's next poll claims them and publishes. Nothing lost.
- **Crash after publish, before `sent_at` is updated.** Same thing. The claim expires and the rows are published **again**. That's why it's at-least-once, and why the consumer must dedupe.
- **Partial batch failure.** 98 of 100 publish fine, 2 fail. Mark only the 98. The 2 wait for their claim to expire and get retried.

### Choosing the claim duration

It must be longer than the slowest realistic publish of a whole batch. If it's too short, a slow pod's claim expires while it's still publishing, a second pod claims the same rows, and both publish. That's still safe (consumers dedupe), just noisy.

### Ordering is not preserved

Ten pods publish ten batches in parallel, and SQS standard queues don't keep order anyway. If order matters for some event type:

- put a sequence number or version in the payload and let the consumer ignore stale updates, or
- use an SQS **FIFO** queue with a message group id (for example, the order id), and accept lower throughput.

### Polling vs CDC

There are two ways to build a relay:

- **Polling** (what we do): one table, one loop. Simple. Cost: latency up to one poll interval (the trace on slide 24 shows a 1.0 s wait).
- **Change data capture (CDC)**, for example Debezium reading the Postgres write-ahead log. Lower latency, no polling load, but more moving parts to run (Kafka Connect, replication slots, and so on).

---

## 6. The queue: SQS [slides 12–13]

### At-least-once delivery

SQS standard queues promise every message is delivered **at least once**. Occasionally a message is delivered twice, even if nothing went wrong. Everything else in the design follows from that.

**Exactly-once delivery across systems isn't something you get from the infrastructure.** What you can build is *effectively once*: at-least-once delivery plus an idempotent consumer. (SQS FIFO queues have deduplication, but only within a 5-minute window and only for the send side, so you still dedupe in the consumer.)

### How a message moves through SQS

1. A consumer calls `ReceiveMessage`. SQS hands it the message and **hides** it from everyone else for the visibility timeout.
2. If the consumer calls `DeleteMessage` (the "ack") in time, the message is gone.
3. If it doesn't (it crashed, threw, or was too slow), the message becomes **visible again** and someone receives it again. The receive count goes up by one.
4. When the receive count passes `maxReceiveCount`, SQS moves the message to the **dead-letter queue (DLQ)**.

### Designing the message [slide 12]

```json
{
  "id": "evt_7c1…",
  "type": "OrderCreated",
  "version": 1,
  "org_id": "org_21a…",
  "occurred_at": "2026-10-10T07:12:00Z",
  "traceparent": "00-4bf9…-01",
  "payload": { "order_id": "…" }
}
```

| Field | Why it's there |
| --- | --- |
| `id` | What the consumer dedupes on |
| `type` + `version` | Consumers can route on type and evolve with versions without breaking |
| `org_id` | The consumer scopes every query to the tenant |
| `occurred_at` | When it happened, not when it was delivered |
| `traceparent` | The trace continues across the async hop [slide 24] |
| `payload` with **ids, not whole objects** | The consumer reads current state from the database. Snapshots go stale, they make messages big, and they spread personal data into queues and logs. |

### The four settings that matter

| Setting | What it does | How to choose |
| --- | --- | --- |
| **Visibility timeout** | How long a received message stays hidden | **Longer than the p99 handling time.** Default is 30 s. For long work, the consumer can extend it while working (`ChangeMessageVisibility`, a "heartbeat"). |
| **`maxReceiveCount`** | Failed receives before moving to the DLQ | The deck uses 5. Too low: a short outage sends good messages to the DLQ. Too high: a poison message wastes work for a long time. |
| **Dead-letter queue** | Where poison messages go | **Alert when it isn't empty.** After you fix the bug, *redrive* the messages back to the main queue (SQS has a redrive feature). |
| **Long polling** | `WaitTimeSeconds = 20` (the maximum) | The receive call waits up to 20 s for a message instead of returning empty right away. Fewer empty receives, lower cost, faster pickup. |

Two extra facts if someone asks:

- Messages stay in a queue for 4 days by default (configurable up to 14).
- For standard queues, a message keeps its *original* enqueue time when it moves to the DLQ. So set the DLQ's retention period longer than the main queue's, or messages can expire in the DLQ sooner than you expect.

### The visibility timeout timeline [slide 13]

- Consumer A receives the message. It's hidden for 30 s.
- A's work takes 45 s.
- At 30 s, the message becomes visible again. Consumer B receives it.
- Without dedupe: **two emails**.

Fix: visibility timeout above p99 handling time, **and dedupe anyway**. Duplicates still happen during deploys, GC pauses and network hiccups.

**Poison message:** a message that always fails (bad data, a bug for one edge case). Receive 1 fails … receive 5 fails → moved to `orders-dlq` → alarm. Without a DLQ it would loop forever, wasting work and hiding real problems.

---

## 7. The consumer [slides 14–15]

### The loop

```kotlin
for (msg in sqs.receive(batch = 10)) {
  try {
    transaction(db) {
      if (!processed.insertIfAbsent(msg.id))
        return@transaction             // already done: skip
      handle(msg)                      // small, scoped by msg.orgId
    }
    sqs.delete(msg)                    // ack only after commit
  } catch (e: Exception) {
    logger.error(e) { "consumer failed id=${msg.id}" }
    // no delete: SQS redelivers after the visibility timeout
  }
}
```

`processed` is a table with the event id as its primary key. `insertIfAbsent` is an `INSERT … ON CONFLICT DO NOTHING` that tells you whether the row was new.

The dedupe row and the work commit **in the same transaction**. If the work fails, the dedupe row rolls back too, so the retry isn't wrongly skipped.

### Why the order matters [slide 15]

| Crash point | Ack first (wrong) | Dedupe, commit, then ack (right) |
| --- | --- | --- |
| Before the work commits | Message already deleted → **lost for good**. No redelivery, no DLQ, no log line. | Transaction rolls back, message redelivered, work runs again. Fine. |
| After commit, before ack | n/a | Message redelivered, dedupe finds the id, skips, acks. Fine. |

> Only ack-first can lose work. Commit-then-ack can only repeat it, and dedupe makes the repeat harmless.

### The four rules [slide 14]

1. **One bad message must not stop the batch.** Catch per message, log with the throwable, move on. Without the per-message `try`, one exception aborts the loop and the other nine messages wait for their visibility timeout.
2. **Keep the transaction small.** No outside calls inside it (same rule as the relay). If handling needs a vendor call, either commit first and call after, or write *another outbox row* and let a worker do it.
3. **Correct with N copies.** Ten pods run this loop at once. Correctness comes from the dedupe table and row locks, not from luck or timing.
4. **Its own executor, its own pool budget.** If the consumer shares the API's thread pool or DB pool, a queue backlog starves the API. Name the executor and count its connections.

**Connection budget:** (API pool + consumer pool + relay pool) per pod × number of pods must stay under the database's `max_connections` (minus room for admin and migrations). Example: (10 + 5 + 2) × 10 pods = 170. If Postgres allows 200, you have very little headroom left for scaling up.

### The tricky case: side effects outside the database (the email)

A sharp intern may spot this. Slide 8 says the consumer does "dedupe, work, commit, ack. Then the email." Think about the crash points:

- Crash **after commit, before the email**: redelivery → dedupe hit → skip → **the email is never sent**.
- Send the email **inside** the transaction, then crash before commit: rollback → redelivery → **second email**.

You can't make an external side effect exactly-once by yourself. You choose:

- **At-least-once** (may send twice): send first, then record. Usually right for emails; a duplicate email is annoying, a missing one is a bug.
- **At-most-once** (may never send): record first, then send.
- **Best of both:** pass a stable **idempotency key to the vendor** (many email and payment APIs accept one, often the event id). The vendor dedupes on its side, so you can retry safely.

The pattern slide 14 recommends is: the consumer writes **another outbox row** ("SendConfirmationEmail"), commits, and a separate worker sends it with retries and the vendor's idempotency key. "If it matters, it's a row" applies again.

---

## 8. When a dependency is slow [slides 16–18]

### Slow is worse than down

A dependency that's **down** fails fast: connection refused in 1 ms. Your code gets an error and moves on.

A dependency that's **slow** holds your threads, connections and memory while you wait. New requests queue up behind the waiting ones. Soon *everything* is slow, including endpoints that never call the vendor. That's how one slow email vendor takes down an order API.

The goal of all four tools: **fail fast and alone.** Fail quickly, and fail only the part that depends on the broken thing.

### Tool 1: timeouts at every hop, each shorter than its caller's

```
client waits 10 s
└─ API gives the vendor call 3 s
   └─ the vendor call gives its connection 1 s
```

If an inner timeout is *longer* than the outer one (for example, the vendor call allows 15 s but the client gives up at 10 s), you keep working for 5 s after the caller has already left: "5 s for nobody" [slide 17]. You pay the cost and nobody gets the result.

Kinds of timeout to know:

- **Connect timeout:** how long to wait to open a TCP connection. Short (around 1 s).
- **Read / socket timeout:** how long to wait between bytes.
- **Request / total timeout:** the whole call, end to end. This is the one that matters most for nesting.
- **Pool acquire timeout:** how long to wait for a free connection from the pool.

**Check your client's defaults.** Many HTTP and database clients default to no timeout or a very long one. "No timeout" is in the top ten mistakes [slide 28].

### Tool 2: retry with backoff, jitter and a budget

- **Limited attempts:** 3 total.
- **Exponential backoff:** wait 200 ms × 2ⁿ (200 ms, 400 ms, 800 ms…).
- **Jitter:** randomize the wait (see section 9).
- **Only retryable errors:** timeouts, `503`, `429` (respect `Retry-After`), connection resets. **Never** `400`, `401`, `404`, `422`: retrying a bad request just repeats the bad request.
- **Only safe-to-repeat calls:** `GET`, or calls with an idempotency key. Retrying a non-idempotent `POST /charge` can charge twice.
- **A budget:** allow retries only while they're under about 10% of calls to that dependency. When the dependency is really down, retries just add load.

### Tool 3: circuit breaker

A state machine around calls to one dependency:

| State | Behavior | Moves to |
| --- | --- | --- |
| **Closed** | Calls flow normally; failures are counted | **Open** after N failures in a time window |
| **Open** | Calls fail **immediately** (return the fallback) | **Half-open** after a cooldown |
| **Half-open** | One probe request is allowed through | **Closed** if the probe succeeds, **Open** again if it fails |

Why it helps: the vendor gets room to recover (you stop hammering it), and your threads aren't tied up waiting for timeouts.

Key detail from slide 17: *while the breaker is open, return the fallback right away. A breaker that waits for a timeout to fail isn't saving anything.*

On the JVM, Resilience4j is the common library for breakers, retries, bulkheads and rate limiters.

### Tool 4: bulkhead (a pool per dependency)

Named after ship compartments: a leak in one compartment doesn't sink the ship.

- **Shared pool of 20 threads:** the vendor hangs, all 20 threads wait on it, `/health` can't get a thread, Kubernetes thinks the pod is dead and **restarts a healthy pod**. Now you have fewer pods and the same problem.
- **Vendor pool of 5 + API pool of 15:** the vendor fills its 5 slots. Further vendor calls fail fast (`503`). `/health` answers in 12 ms. Everything else keeps serving.

### Little's law: how big should a pool be?

```
L = λ × W
in flight = arrival rate × time per request
```

- 50 req/s × 200 ms = **10** in flight. A pool of 20 is comfortable.
- 50 req/s × 3 s (slow vendor) = **150** in flight. A pool of 20 is overwhelmed.

Little's law holds for any stable system, whatever the traffic pattern. It's the quickest way to check whether a pool size makes sense. The number explains the bulkhead: when the vendor slows from 200 ms to 3 s, demand for threads goes up 15×. That has to come out of a separate budget.

Slide 18 has a live calculator. Practice dragging the sliders before the session.

---

## 9. The fine print [slides 19–20]

### Retry amplification

Three layers (client → API → service → vendor), each making up to 3 attempts: 3 × 3 × 3 = **27 calls** on the vendor from one click, at exactly the moment the vendor is struggling. If each layer does "3 retries" (4 attempts), it's 4³ = 64.

**Retry in one place**, usually the layer closest to the dependency. The other layers pass the error up.

### Propagate the deadline, not a fixed timeout

If the client gave you 10 s and 7 s are already gone, the vendor call should get the remaining 3 s minus a margin, not its usual 3 s. Pass the deadline down (a header like `X-Request-Deadline`, or gRPC deadlines, which do this for you) and check it before each retry. No point starting a retry that can't finish in time.

### Decide what failing fast returns

An error is the last option, not the default. Options, from best to worst:

1. **Cached or stale answer:** yesterday's exchange rate is better than none.
2. **Reduced feature:** show the page without recommendations.
3. **"Accepted, we'll finish later":** return `202 Accepted` and write an outbox row. This links back to the start of the session.
4. **An error**, with a clear message.

### Jitter [slide 20]

**The thundering herd:** 200 clients fail at the same moment (the vendor blipped). With plain exponential backoff, all 200 retry at exactly 200 ms, then exactly 600 ms, then exactly 1.4 s (cumulative). Each retry wave hits the vendor with the full 200 at once and knocks it over again.

**Full jitter:** wait `random(0, base × 2ⁿ)` instead of exactly `base × 2ⁿ`. The same 600 retries spread across the window, and the peak drops far below the vendor's capacity. (The slide computes this in the browser, so the exact peak number changes each time you load it.)

The classic reference is the AWS Architecture Blog post "Exponential Backoff and Jitter".

---

## 10. Backpressure [slide 21]

### The knee

Latency stays flat as load grows, until some resource saturates (here the DB pool of 20, at about 240 req/s). After the knee, requests queue up and latency goes nearly vertical. Queueing theory says the same thing: as utilization approaches 100%, waiting time grows without bound.

### Unbounded queues are a trap

An unbounded in-memory queue (an executor with an unlimited work queue, an unlimited channel) hides overload. It just grows, latency grows, then the pod runs out of memory and **everything in the queue is lost at once**.

### Say no early

- **Bounded executors and bounded queues.**
- When full, reject immediately with **`429 Too Many Requests`** or **`503 Service Unavailable`**, plus a **`Retry-After`** header.
- The rejected requests fail in milliseconds. The accepted ones stay fast. p99 stays flat.

> A fast no is cheaper for everyone than a slow yes that times out.

`429` usually means "you, specifically, are sending too much" (rate limit). `503` means "the service is overloaded right now". Either one with `Retry-After` tells well-behaved clients when to come back.

### Find your knee

Load-test with **k6** (or a similar tool): ramp traffic up, watch p99, find where it jumps, and name the limit that caused it (DB pool, CPU, a vendor). That number goes in the runbook. The slide's curve is illustrative; replace it with a real run if you have one.

---

## 11. Graceful shutdown [slide 22]

**Every deploy is a shutdown.** If shutdown isn't clean, every deploy drops some requests.

### What happens when Kubernetes stops a pod

The precise sequence (slightly more detailed than the slide):

1. The pod is marked **Terminating**. In parallel, Kubernetes starts removing it from Service endpoints, so load balancers stop sending it new traffic. This removal is *asynchronous* and takes a few seconds to reach every node and load balancer.
2. The **`preStop` hook** runs, if there is one. A `sleep 5` here gives the endpoint removal time to spread, so the pod doesn't stop accepting connections while traffic is still arriving.
3. The container receives **`SIGTERM`**. Your app starts its shutdown.
4. If the container is still running when **`terminationGracePeriodSeconds`** (default **30 s**, counted from the start, including the preStop time) runs out, it gets **`SIGKILL`**. No more code runs.

The slide draws `SIGTERM` and the preStop sleep at the same starting point to keep the picture simple. If someone asks, the precise order is: preStop first, then SIGTERM to the process.

### What each loop does on shutdown

| Loop | On shutdown |
| --- | --- |
| Readiness | Report not ready (helps with traffic that still arrives) |
| HTTP server | Stop accepting new connections, finish in-flight requests, then stop |
| SQS consumer | Stop calling `ReceiveMessage`, finish the current batch, ack what committed |
| Outbox relay | Stop claiming, publish what's already claimed, mark it sent |
| DB pool / process | Close the pool, `exit 0` |

In the slide's example, everything finishes at **9.5 s**, well before SIGKILL at 30 s.

### What's lost if you're still running at 30 s

- **Dropped HTTP requests:** clients see errors.
- **Half-processed messages:** the transaction never committed, so it rolls back. The message is redelivered after its visibility timeout. Dedupe saves you.
- **Relay claims:** sit until `claimed_until` expires, then another pod picks them up. Delayed, not lost.

The outbox and dedupe design means a hard kill **delays** work but doesn't **lose** it. Graceful shutdown makes deploys smooth instead of noisy.

---

## 12. Observability [slides 23–24]

### Three signals, three questions

| Signal | Answers | Key rules |
| --- | --- | --- |
| **Logs** | What happened to *this* request? | Structured JSON, one event per line. Every line carries `request_id`, `org_id`, `user_id` (from the MDC). Errors log the throwable (stack trace). No tokens, no PII you don't need. |
| **Metrics** | Is the service healthy *right now*? | Rate, errors, duration (p50/p95/p99) per endpoint and per consumer. Plus saturation: pool in use, executor queue depth, queue age, DLQ size, **outbox lag**. |
| **Traces** | *Where* did the time go? | One trace id across API, DB, vendor call and consumer. OpenTelemetry. Propagated in HTTP headers **and in the message**. |

### Terms interns may not know

- **MDC (Mapped Diagnostic Context):** a per-request (per-thread or per-coroutine) map that the logging library adds to every log line automatically. Set `request_id` once at the start of the request; every log line after that includes it.
- **RED:** Rate, Errors, Duration. The standard trio for request-driven services.
- **Saturation:** how full a resource is (pool 18/20 in use). The early warning before latency jumps. Google's "four golden signals" are latency, traffic, errors, saturation, which is RED plus saturation.
- **Percentiles:** p99 = 99% of requests are faster than this. Use percentiles, not averages. An average of 100 ms can hide 1% of users waiting 10 s.
- **Span:** one timed operation inside a trace (an HTTP call, a DB query). A trace is a tree of spans.
- **`traceparent`:** the W3C Trace Context header, `version-traceid-spanid-flags`. Putting it in the SQS message attributes makes the consumer's spans join the API's trace.

### Page on symptoms, not causes

Pages (alerts that wake someone up):

- Error rate over **1% for 5 minutes**
- p99 over the SLO
- DLQ not empty
- Queue age (oldest message) over **10 minutes**
- **Oldest unsent outbox row over 5 minutes**

Not "CPU is 80%". High CPU might be fine. Users being hurt is never fine.

### Every alert has a runbook and a person

An alert nobody acts on gets muted within a month, and then it's worse than no alert, because people think they're covered. Fewer alerts, each one real. Once you have SLOs, **burn-rate alerts** ("we're spending our monthly error budget 10× too fast") beat fixed thresholds. The Google SRE Workbook chapter on alerting on SLOs is the reference.

### The outbox lag metric

```sql
SELECT now() - min(created_at) AS outbox_lag
  FROM outbox
 WHERE sent_at IS NULL;
```

Cheap with the partial index on unsent rows. Export it as a gauge and alert when it's over 5 minutes.

### One trace, end to end [slide 24]

Where the 4.2 seconds went (illustrative):

| Span | Time |
| --- | --- |
| `api · POST /orders` (user gets 201 here) | 180 ms |
| Wait for the relay's next poll | 1.0 s |
| `relay · claim + publish` | 60 ms |
| `sqs · in queue` | 30 ms |
| `consumer · OrderCreated` | 2.91 s |
| └ `email-vendor · POST /send` | **2.83 s** ← the time went here |

Without `traceparent` in the message, this would be two unrelated traces and a guess.

---

## 13. The 3 a.m. incident [slides 3 and 26]

### The setup [slide 3]

- 03:07. Every dashboard is green. Customers say they never got their confirmation email.
- The graph: **SQS queue depth** (alerted) is flat near zero all night. **Outbox rows with `sent_at IS NULL`** (not alerted) start climbing at 02:10, right after a deploy.
- No page fired. The queue depth looks perfect.

### The answer [slide 26]

- The **relay stopped at the 02:10 deploy** (the slides don't say why: a config change, a crashed loop that nobody restarted, a disabled scheduler…).
- The queue is empty **because nothing is being published**. An empty queue looks exactly like a healthy queue. All the queue alerts (depth, age, DLQ) stay quiet.
- Only the outbox table shows the problem.
- With the outbox-lag alert ("oldest unsent row > 5 min"), someone gets paged at **02:15**. Customers noticed at **03:07**. That's **52 minutes earlier**.

### The lesson

> The fix was one query and one alert rule. The lesson is the pattern: every async hop needs its own "is work piling up here?" signal.

For this pipeline, that's: outbox lag (relay), queue age (consumer), DLQ size (poison messages).

The incident is illustrative. If you have a real one from your team, use it; real stories stick much better.

---

## 14. Recap material [slides 27–28]

The **nine numbers** [slide 27] and **ten mistakes** [slide 28] are covered slide by slide in the script. All of them come from earlier slides, so if you understand sections 1–13, you can explain every one. The numbers are examples; adjust them to your service's real values if you know them.

---

## 15. Further reading

- Michael Nygard, *Release It!* (2nd edition). Timeouts, circuit breakers, bulkheads, steady state. The book the deck points to.
- microservices.io, "Pattern: Transactional outbox" (Chris Richardson).
- AWS Architecture Blog, "Exponential Backoff and Jitter" (Marc Brooker).
- AWS docs: Amazon SQS visibility timeout, dead-letter queues, long polling.
- Stripe API docs, "Idempotent requests"; IETF draft "The Idempotency-Key HTTP Header Field".
- Google SRE Book, "Monitoring Distributed Systems" (four golden signals); SRE Workbook, "Alerting on SLOs" (burn rates).
- Kubernetes docs, "Pod Lifecycle → Termination of Pods".
- W3C Trace Context specification (`traceparent`).
