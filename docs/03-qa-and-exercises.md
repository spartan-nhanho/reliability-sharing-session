# Q&A, exercises and quiz

Material to support the 2-hour session: questions interns are likely to ask (with answers), in-session exercises with answer keys, a take-home quiz, and a glossary.

---

## Part A · Questions interns are likely to ask

Short answers you can give out loud. The study guide (`01-presenter-study-guide.md`) has the longer versions.

### Dual writes and the outbox

**Q: How often does a pod really die between two lines of code?**
Rarely per request, but often in total. Every rolling deploy kills every old pod; OOM kills and node drains happen too. At thousands of requests a day, "rare" becomes "a few customers every week".

**Q: Why not use a distributed transaction (two-phase commit)?**
SQS and email vendors don't support it. Where it exists, it's slow, holds locks across the network, and can get stuck if the coordinator fails. Making the database the single source of truth is simpler and more reliable.

**Q: Isn't the outbox just moving the problem? The relay still has to send to SQS.**
Yes, but now the send can be *retried safely*. The row stays in the outbox until it's marked sent, so a crash only delays the send. Before, a crash meant the event was forgotten.

**Q: Why not just retry the `sqs.send` in the request if it fails?**
Retrying helps with a failed send, not with a dead pod. If the pod is killed after the commit, there's no code left running to retry.

**Q: Doesn't the outbox make everything slower?**
The request does one more small insert in a transaction it was already running, so the request gets barely slower. The event arrives a little later (up to one poll interval, about 1 s in the trace on slide 24). For emails and notifications, that's fine. If you need lower latency, use CDC instead of polling.

**Q: What's in the payload? Why only ids?**
The consumer reads current state from the database. A full copy of the order in the message would be stale by the time it's processed, makes messages bigger, and spreads personal data into queues and logs.

### Idempotency

**Q: Who generates the idempotency key, the client or the server?**
The client. Only the client knows that two requests are "the same tap of the button". It generates the key once per user intent and reuses it on every retry.

**Q: What if the client reuses a key with a different request body?**
That's a client bug. Store a hash of the request with the key; if a retry has the same key but a different body, return an error (`422` or `409`) instead of the old response.

**Q: What if two requests with the same key arrive at the same moment?**
Both look up the key and find nothing. Both try to insert. The unique constraint on `(org_id, key)` makes the second insert wait until the first commits, then fail. The second request catches the error, reads the stored response and returns it.

**Q: If we have idempotency keys, do we still need consumer dedupe?**
Yes. They fix different duplicates. Keys stop the *API* from creating two events for one user action. Consumer dedupe handles *SQS* delivering one event twice.

### Relay and SQS

**Q: What does `SKIP LOCKED` do exactly?**
`FOR UPDATE` locks the selected rows. `SKIP LOCKED` means: if another transaction has already locked a row, skip it instead of waiting. That way ten relay pods each grab a different batch without blocking each other.

**Q: Why is holding a lock during the SQS call so bad if it's only 20 ms normally?**
Because "normally" isn't the day that hurts. On the day SQS takes 1 s, every relay pod holds a DB connection and row locks for that second. Connection pools run out, the API waits, the cleanup job is blocked. The design has to survive the bad day.

**Q: What happens if the claim expires while a pod is still publishing?**
Another pod may claim and publish the same rows, so a message is sent twice. That's still safe because consumers dedupe. It's just noisy. Set the claim longer than the slowest realistic batch publish.

**Q: Is SQS FIFO exactly-once? Can we skip dedupe then?**
No. FIFO deduplicates sends only within a 5-minute window, and a consumer can still crash after doing the work but before deleting the message, which causes a redelivery. Always dedupe in the consumer.

**Q: What's the difference between visibility timeout and message retention?**
Visibility timeout: how long a message stays hidden after one consumer receives it (default 30 s). Retention: how long a message can stay in the queue at all before SQS deletes it (default 4 days, up to 14).

**Q: What if the work sometimes takes longer than the visibility timeout?**
Either set the timeout above the p99 handling time, or have the consumer extend the timeout while it works (`ChangeMessageVisibility`, a "heartbeat"). And dedupe anyway.

**Q: What do we do with messages in the DLQ?**
The DLQ alarm pages someone. They look at the messages to find the bug, fix it and deploy, then *redrive* the messages back to the main queue so they're processed. Never just delete them without understanding why they failed.

**Q: How do I choose `maxReceiveCount`?**
High enough that a short outage (a vendor down for a minute) doesn't send good messages to the DLQ. Low enough that a truly broken message doesn't waste work for hours. The deck uses 5; together with the visibility timeout, it defines how long a message keeps retrying.

### Consumer

**Q: Why put the dedupe insert and the work in the same transaction?**
If the work fails and rolls back, the dedupe row rolls back too. Otherwise the id would be marked "processed" even though the work never happened, and the retry would skip it. The work would be lost.

**Q: The email isn't in the database. How can it be "exactly once"?**
It can't, on its own. You choose: send then record (may send twice), or record then send (may never send). Usually a duplicate email is better than a missing one. Even better: pass the event id as an idempotency key to the email vendor, or write a "send email" outbox row and let a separate worker send it with retries.

**Q: Why does the consumer need its own thread pool?**
If it shares the API's pool, a big queue backlog can take all the threads, and API requests wait. A separate pool limits the damage to the consumer.

### Slow dependencies

**Q: How do I pick timeout values?**
Start from the outside: what the client will wait (for example 10 s). Each inner call gets less, leaving room for your own work and maybe one retry. Then check against real latency: a timeout well above the dependency's normal p99, but well below the caller's timeout.

**Q: Which errors should I retry?**
Errors that might succeed next time: timeouts, connection resets, `503`, `429` (wait for `Retry-After`). Not `400`, `401`, `403`, `404`, `422`: the request itself is wrong, so it'll fail the same way again.

**Q: What's the difference between a circuit breaker and a retry?**
A retry tries again after one failure. A breaker stops trying after many failures. They work together: retry a little for blips; when the dependency is clearly down, the breaker opens and stops the retries.

**Q: What's a good fallback when the breaker is open?**
A cached or stale value, a reduced feature (hide recommendations), or "accepted, we'll do it later" (`202` + outbox row). An error is the last resort.

**Q: Isn't a bulkhead wasteful? Threads sitting idle in the vendor pool?**
A few idle threads cost very little. A whole service going down because one vendor is slow costs a lot more.

**Q: Where does Little's law come from? Does it work with bursty traffic?**
It's a theorem from queueing theory: in any stable system, average number in the system = average arrival rate × average time in the system. It holds whatever the traffic pattern. For bursty traffic, size for the busy periods, not the daily average.

**Q: What's the difference between 429 and 503?**
`429 Too Many Requests` usually means "you are sending too much" (a rate limit for that caller). `503 Service Unavailable` means "the service is overloaded or down right now". Send `Retry-After` with either.

### Shutdown and observability

**Q: What happens to a message the consumer is working on when the pod gets SIGKILL?**
The transaction never commits, so it rolls back. The message wasn't deleted, so after its visibility timeout SQS delivers it again. The work is delayed, not lost.

**Q: What is the MDC?**
Mapped Diagnostic Context: a per-request map the logging library attaches to every log line. Set `request_id` and `org_id` once at the start of the request, and every log line automatically includes them. With coroutines, the context has to be passed along explicitly (for example with the `MDCContext` element in kotlinx-coroutines-slf4j).

**Q: Why percentiles and not averages?**
Averages hide the slow tail. If 99 requests take 10 ms and 1 takes 10 s, the average is about 110 ms, which looks fine. The p99 shows the 10 s that real users waited.

**Q: Why isn't "CPU over 80%" a good page?**
It's a cause, not a symptom. CPU can be high while users are perfectly fine, and users can be suffering while CPU is low (for example, when everything is waiting on a slow vendor). Page on what users feel; use CPU on dashboards to investigate.

**Q: Why didn't the queue alerts catch the 3 a.m. incident?**
Because the relay stopped, nothing was published, so the queue was empty. An empty queue looks exactly like a healthy one. Only the outbox table showed work piling up.

---

## Part B · In-session exercises

### Exercise 1 · Draw the pipeline (after slide 15, 3 minutes)

**Instructions for interns (in pairs):**

From memory, draw the path of one order event, from the client to the email. For each box, write one thing that can go wrong and what protects against it.

**Answer key:**

| Box | What can go wrong | What protects |
| --- | --- | --- |
| Client → API | Client times out and retries | Idempotency key |
| API → Postgres | Pod dies after commit | Outbox row in the same transaction |
| Relay | Crash after publish, before mark sent | Claim expires, row republished (at-least-once) |
| Relay | SQS is slow | Claim, commit, then publish (no locks held) |
| SQS | Delivers the same message twice | Consumer dedupe by event id |
| Consumer | Work takes longer than the visibility timeout | Timeout above p99 + dedupe |
| Consumer | Crash before ack | Commit, then ack + dedupe |
| Consumer | Message always fails | `maxReceiveCount` → DLQ + alarm |
| Email vendor | Slow | Timeout, retry, breaker, bulkhead |
| Relay (whole) | Stops completely | Outbox lag alert |

### Exercise 2 · Where can it crash? (optional, after slide 15, 4 minutes)

**Instructions:** For each crash point, say what happens to the work. Is it lost, done twice, or fine?

```kotlin
val msg = sqs.receive()                  // A
transaction(db) {
  if (!processed.insertIfAbsent(msg.id)) return
  handle(msg)                            // B
}                                        // C (commit)
sqs.delete(msg)                          // D
                                         // E (after delete)
```

**Answer key:**

| Crash at | What happens |
| --- | --- |
| After A, before B | Nothing committed. Message redelivered after the visibility timeout. Work runs once. **Fine.** |
| During B | Transaction rolls back (including the dedupe row). Redelivered. Work runs once. **Fine.** |
| After C, before D | Work committed. Message redelivered, dedupe finds the id, skips, deletes. **Fine (repeat is a no-op).** |
| After D | Everything done. **Fine.** |

Follow-up question: *"Now move `sqs.delete(msg)` to before the transaction. Where does it break?"*
Answer: crash after the delete, before the commit → the message is gone and the work never happened. **Lost.**

### Exercise 3 · Little's law by hand (optional, after slide 18, 3 minutes)

**Instructions:** in flight = rate × time. Calculate, then say whether a pool of 20 is enough.

| # | Rate | Time per request | In flight | Pool of 20 OK? |
| --- | --- | --- | --- | --- |
| 1 | 100 req/s | 50 ms | ? | ? |
| 2 | 40 req/s | 500 ms | ? | ? |
| 3 | 40 req/s | 2 s | ? | ? |
| 4 | 200 req/s | 100 ms | ? | ? |

**Answer key:** 1 → 5, yes. 2 → 20, just at the limit (no headroom; any slowdown queues). 3 → 80, no. 4 → 20, at the limit.

Bonus: *"Our vendor normally takes 200 ms at 50 req/s. It slows to 3 s. How many threads does it need now, and what happens to everything else if it shares a pool of 20?"*
Answer: 150. It takes all 20 threads, and every other endpoint waits. That's why it gets a bulkhead.

### Exercise 4 · Review this code (alternative to the live part, 10–14 minutes)

Use this if you can't show the real repo. Put each snippet on screen or on paper. Ask: *"What's wrong, and how would you fix it?"* Each snippet has one or more mistakes from slide 28.

**Snippet 1**

```kotlin
fun createOrder(req: CreateOrderRequest): Order {
  val order = transaction(db) { orders.insert(req.toOrder()) }
  GlobalScope.launch {
    emailClient.sendConfirmation(order)
  }
  return order
}
```

*Answer:* **launch { }** and **dual write**. The email lives only in a coroutine; if the pod dies, it's lost. Fix: insert an outbox row (`OrderCreated` or `SendConfirmationEmail`) in the same transaction, and let a worker send it.

**Snippet 2**

```kotlin
for (msg in sqs.receive(batch = 10)) {
  sqs.delete(msg)
  transaction(db) { handle(msg) }
}
```

*Answers:* **ack first**: a crash after the delete loses the message. **batch stops**: no per-message try/catch, so one exception stops the other nine. **no dedupe**: a redelivered message is processed twice. Fix: the consumer loop from slide 14.

**Snippet 3**

```kotlin
transaction(db) {
  val rows = outbox.selectUnsentForUpdate(limit = 100)
  rows.forEach { sqs.send(it.toMessage()) }
  outbox.markSent(rows.map { it.id })
}
```

*Answer:* a **row lock held across a network call**. When SQS is slow, connections and locks are held for the whole publish. Fix: claim with `claimed_until` and `SKIP LOCKED`, commit, publish, then mark sent (slide 10).

**Snippet 4**

```kotlin
val client = HttpClient(CIO)            // no timeout configured

suspend fun charge(req: ChargeRequest) =
  retry(times = 5) {                     // retries on any exception
    client.post("https://payments.example/charge") { setBody(req) }
  }
```

*Answers:* **no timeout**: the code relies on whatever the engine's default is, which nobody checked. Defaults are often missing or longer than the caller's 10 s, so the call keeps working "for nobody" when the vendor hangs. **retry everywhere / unsafe retry**: retrying a non-idempotent `POST /charge` on any exception can charge twice, and it also retries `400`s. No backoff or jitter. Fix: set connect and request timeouts; retry only timeouts / `503` / `429`, with backoff and jitter; send an idempotency key to the payment vendor.

**Snippet 5**

```kotlin
logger.error("Something failed")
```

*Answer:* **no context**. No request id, org id, message id, or exception. Fix: `logger.error(e) { "consumer failed id=${msg.id}" }`, with `request_id` / `org_id` in the MDC.

**Snippet 6 (Terraform)**

```hcl
resource "aws_sqs_queue" "orders_events" {
  name                       = "orders-events"
  visibility_timeout_seconds = 10
}
```

The consumer's p99 handling time is 25 s.

*Answers:* the **visibility timeout** (10 s) is shorter than the work (25 s), so messages will be delivered twice under normal load. **No DLQ** (no `redrive_policy`), so poison messages loop forever. **No long polling** (`receive_wait_time_seconds`). Fix: timeout above p99 (for example 60 s), add a DLQ with `maxReceiveCount = 5` and an alarm on it, set `receive_wait_time_seconds = 20`.

---

## Part C · Take-home quiz

Ten questions. Share after the session, or use the first five as a quick check before Q&A.

1. What are the four possible outcomes of a dual write, and which two are bugs?
2. Why doesn't swapping the order of "commit" and "send" fix the dual-write problem?
3. A mobile app retries `POST /orders` after a timeout. The first request had already committed. Why doesn't consumer dedupe by event id prevent a double charge? What does?
4. In the relay, why do we commit the claim *before* publishing to SQS?
5. What does `FOR UPDATE SKIP LOCKED` let us do?
6. SQS visibility timeout is 30 s and the consumer takes 45 s. What happens? Name two fixes.
7. Why is "ack, then do the work" dangerous, but "do the work, commit, then ack" safe?
8. Three layers each make 3 attempts on a failing call. How many calls reach the vendor from one click? What's the fix?
9. 80 req/s, each call takes 250 ms. How many requests are in flight? Is a pool of 15 enough?
10. In the 3 a.m. incident, why did no queue alert fire, and which alert would have caught it?

### Answer key

1. Happy path, nothing happened (honest); **lost event** (commit OK, send failed) and **ghost event** (send OK, commit failed) are bugs.
2. It only swaps which bug you get: send first → ghost event; commit first → lost event. You need a single source of truth: the outbox.
3. The retry created a *second order with a different event id*, so the consumer sees two different ids. An **idempotency key** at the API dedupes the user's intent.
4. So no row locks or DB connections are held during the network call. If SQS is slow, the database isn't blocked. The `claimed_until` lease protects the rows instead.
5. Many relay pods can each claim a different batch at the same time without waiting on each other's locks.
6. At 30 s the message becomes visible again and a second consumer processes it: duplicate work (for example, two emails). Fixes: visibility timeout above p99 handling time (or extend it while working), and dedupe in the consumer.
7. Ack first: a crash after the ack loses the message forever. Commit then ack: a crash before the ack only causes a redelivery, and dedupe turns it into a no-op.
8. 3 × 3 × 3 = **27**. Retry at one layer only, usually the one closest to the dependency.
9. 80 × 0.25 = **20** in flight. A pool of 15 is not enough: requests will queue.
10. The relay had stopped, so nothing was published and the queue stayed empty, which looks healthy. An **outbox lag** alert (oldest unsent row > 5 minutes) would have paged at 02:15, 52 minutes before customers noticed.

---

## Part D · Glossary

| Term | Meaning |
| --- | --- |
| **Ack** | Acknowledging a message so the queue removes it. In SQS, `DeleteMessage`. |
| **At-least-once** | Every message is delivered one or more times. Duplicates are possible; loss is not. |
| **At-most-once** | Every message is delivered zero or one time. Loss is possible; duplicates are not. |
| **Backoff** | Waiting longer between each retry (for example 200 ms, 400 ms, 800 ms). |
| **Backpressure** | Pushing back on callers when you're full (for example a fast `429`) instead of accepting unlimited work. |
| **Bulkhead** | A separate, limited pool (threads or connections) per dependency, so one slow dependency can't use up everything. |
| **CDC** | Change data capture. Reading the database's change log (for example with Debezium) to publish events. |
| **Circuit breaker** | Stops calling a failing dependency for a cooldown, then tests it with one probe. States: closed, open, half-open. |
| **Claim / lease** | Marking rows as "being worked on until time T" (`claimed_until`), so a crashed worker's rows can be picked up after T. |
| **Consumer** | The worker that receives messages from the queue and does the work. |
| **Dead-letter queue (DLQ)** | A queue where messages go after failing too many times. |
| **Dedupe** | Detecting and skipping work already done, usually with a table of processed ids. |
| **Dual write** | Writing to two systems (for example DB and queue) without a shared transaction. |
| **Effectively once** | At-least-once delivery + idempotent processing = each effect happens once. |
| **Idempotent** | Doing it twice has the same effect as doing it once. |
| **Idempotency key** | A client-generated id, one per user intent, used by the API to recognize retries. |
| **Jitter** | Randomness added to retry waits so clients don't retry at the same moment. |
| **Little's law** | In flight = arrival rate × time per request. |
| **Long polling** | A receive call that waits (up to 20 s in SQS) for a message instead of returning empty right away. |
| **`maxReceiveCount`** | SQS setting: how many receives before a message moves to the DLQ. |
| **MDC** | Mapped Diagnostic Context. Per-request fields automatically added to every log line. |
| **OOM kill** | The kernel killing a process that went over its memory limit. No cleanup runs. |
| **Outbox** | A table of events written in the same transaction as the business data, published later by a relay. |
| **Outbox lag** | Age of the oldest unsent outbox row. Shows whether the relay is keeping up. |
| **p50 / p95 / p99** | Percentiles. p99 = 99% of requests are faster than this. |
| **Poison message** | A message that fails every time it's processed. |
| **preStop hook** | A Kubernetes hook that runs before the container gets SIGTERM, often a short sleep. |
| **RED** | Rate, Errors, Duration: the basic metrics for a service. |
| **Redrive** | Moving messages from the DLQ back to the main queue after a fix. |
| **Relay** | The loop that reads unsent outbox rows and publishes them to the queue. |
| **Runbook** | A written guide for what to do when a specific alert fires. |
| **Saturation** | How full a resource is (pool in use, queue depth). |
| **SIGTERM / SIGKILL** | "Please stop" (your code can react) / "stop now" (it can't). |
| **SLO** | Service level objective: a target such as "99.9% of requests succeed in under 500 ms". |
| **Span / trace** | A span is one timed operation; a trace is the tree of spans for one request. |
| **`traceparent`** | W3C header carrying the trace id, so a trace continues across services and queues. |
| **Visibility timeout** | How long SQS hides a message after a consumer receives it. |
