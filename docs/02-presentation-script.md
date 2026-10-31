# Presentation script: 2-hour intern session

A slide-by-slide script for teaching the deck to interns in about 2 hours. The deck was written for 60 minutes. This version slows down and adds checks for understanding, a break, and short exercises.

For the background behind each slide, read `01-presenter-study-guide.md`. For questions, exercises and the quiz, see `03-qa-and-exercises.md`.

## How to read this script

Each slide has:

- **Time:** minutes for this slide, and the clock time you should be at when you *leave* it.
- **Clicks:** what each `→` press reveals. (Press `N` in the deck to see the built-in speaker notes.)
- **Say:** a script in plain spoken English. Don't read it word for word; say it in your own words. Lines in *italics* are worth saying exactly.
- **Ask:** questions for the room, with the answers you're hoping for.
- **Next:** a one-line transition to the next slide.

---

## Before the session

### The day before

- [ ] Read the study guide once, end to end.
- [ ] Click through the deck twice with `N` (notes) open. Practice the build steps until you know what each `→` reveals.
- [ ] Practice the Little's law sliders on slide 18 and the jitter chart on slide 20.
- [ ] Decide what you'll show in the live part (slide 25). Fill in the file paths in the "Live walkthrough" section below.
- [ ] If you have a real incident from your team, prepare it as a replacement for the 3 a.m. story.
- [ ] Optional: slide 2 says "15 min / 30 min / 15 min" (the 60-minute version). Either edit it, or just tell the room "we're taking it slower today".

### 30 minutes before

- [ ] Open the deck in Chrome, full screen (`F`). Check that the font loaded.
- [ ] Open tabs for the live part: the repo (outbox write, relay, consumer), Terraform for the queue, and the dashboards.
- [ ] Open a whiteboard or paper for drawings.
- [ ] Print or share the quiz from `03-qa-and-exercises.md` if you plan to use it.
- [ ] Have water.

### Running order

| Block | Slides | Minutes | Clock at end |
| --- | --- | --- | --- |
| Opening and the 3 a.m. hook | 1–3 | 7 | 0:07 |
| Part 1 · The dual write and idempotency | 4–7 | 15 | 0:22 |
| Part 2 · The pipeline: outbox, relay, queue, consumer | 8–15 | 28 | 0:50 |
| Exercise 1 + break | — | 7 | 0:57 |
| Part 3 · Slow dependencies | 16–22 | 24 | 1:21 |
| Part 4 · Observability | 23–24 | 8 | 1:29 |
| Live walkthrough | 25 | 14 | 1:43 |
| The answer, recap, review checklist | 26–28 | 7 | 1:50 |
| Q&A and close | 29 | 10 | 2:00 |

### If you're running late or early

**Behind schedule? Cut these first** (in order):

1. Slide 11 (lock time): say the one-line takeaway and skip the build.
2. Slide 20 (jitter): show it, say one sentence, move on.
3. Shorten the live part to steps 1, 3 and 6.
4. Slide 22 (shutdown): explain the table only.

**Never cut:** slides 4–5 (dual write), 9 (outbox), 15 (ack order), 16 (four tools), 23 (observability), 26 (the answer).

**Ahead of schedule? Add these:**

- Exercise 2 (crash points) after slide 15.
- Exercise 3 (Little's law by hand) after slide 18.
- Let the room drive the incident replay longer in the live part.

### Tips for teaching interns

- **Ask, then wait.** After a question, count to five silently before you help. Interns need time to be brave.
- **Use names gently.** "Anyone who hasn't spoken yet: what would you check first?"
- **Praise wrong answers that reason well.** "Good thinking, that's exactly what most people try first. Let's see why it breaks."
- **Draw.** Whenever a question is about order of operations, draw boxes and arrows on the whiteboard.
- **Repeat questions** before answering so everyone hears them.

---

## Opening (0:00–0:07)

### Slide 1 · Title

**Time:** 2 min · leave at 0:02

**Say:**

> Hi everyone. I'm [name], I work on [team]. Today is about reliability: what happens to work after a request finishes, what happens when something we depend on gets slow, and how we find out what went wrong at 3 in the morning.
>
> The plan: first ideas, with diagrams. Then I'll show you the real code and dashboards. At the end, the mistakes code review catches most, and questions.
>
> Two things. First, please ask questions any time. Interrupt me. There are no stupid questions today; if you're confused, someone else is too. Second, many slides build step by step. When I press the arrow, the next part appears.
>
> We have about two hours, with a short break in the middle.

**Next:** "Let me start with why this topic matters."

### Slide 2 · Why this session

**Time:** 2 min · leave at 0:04

**Say:**

> Read the headline with me. *Most outages aren't bugs in the code. They're the code meeting a slow dependency it never planned for.*
>
> When you write a feature, you test it on your laptop. The database answers in 2 milliseconds, the email service always works, nothing crashes. Production isn't like that. Pods get killed during deploys. Vendors slow down. Networks drop packets. The code is "correct", but it never planned for any of that.
>
> Today has three parts. The concepts: how work leaves a request without being lost or doubled, and how to fail fast. Then the real pipeline and the dashboards. Then what code review catches, and questions.

(If you didn't edit the minute labels: "The minutes on the slide are for the short version. Today we're going slower, with exercises.")

**Next:** "Let's start with a mystery."

### Slide 3 · 03:07 a.m.

**Time:** 3 min · leave at 0:07

**Clicks:** on entry, the blue queue line draws, the deploy marker appears at 02:10, then the red outbox line climbs. `→` reveals "No page fired" and "Hold that thought".

**Say:**

> It's 3:07 in the morning. Every dashboard is green. But customers are writing in: "I never got my confirmation email."
>
> Look at the graph. Blue is the SQS queue depth, the number of messages waiting in the queue. We have an alert on it. It's flat, near zero, all night. That looks healthy.
>
> There's a deploy at 2:10. And red is something else: rows in a table called `outbox` that haven't been sent yet. After the deploy, it starts climbing. There's no alert on that.

**Ask:** *"You're on call. What would you check first?"* Take two or three guesses. **Don't answer.** Write the guesses on the whiteboard if you can; you'll come back to them.

Likely guesses: "the email service is down", "the deploy broke something", "check the logs". All reasonable.

`→`

> No page fired. The queue depth looks perfect. Hold that thought. By the end of today, you'll be able to explain this graph yourself.

**Next:** "To understand it, we need to start with a very common piece of code."

---

## Part 1 · The dual write and idempotency (0:07–0:22)

### Slide 4 · Two systems, one request, no shared transaction

**Time:** 4 min · leave at 0:11

**Clicks:** `→` ×3 reveals the three rules one by one.

**Say:**

> Here's code almost everyone writes the first time. Inside a database transaction, we insert the order. Then, after the transaction, we send an `OrderCreated` message to SQS, and we send a confirmation email.

**Ask:** *"Where can this break?"* Wait. Point at the `// ← here` comment if nobody answers.

Good answers: "the pod could crash after the commit", "SQS could be down", "the send could fail".

> Right. The pod can die between the commit and the send. Maybe it's a deploy, maybe it ran out of memory. The order is saved, but the event is never sent and the email never goes out. And nothing tells us. No error, no log line. The customer just never hears from us.
>
> You might think: put the send *inside* the transaction. But then the send succeeds, and the commit fails. Now we've sent an email for an order that doesn't exist.
>
> The problem is: Postgres and SQS are two different systems. There's no transaction that covers both.

**Analogy (optional):** "It's like telling a friend 'I paid for dinner' before your card payment goes through. If the payment fails, your friend already believes something that's not true."

> So here are the rules for what a request is allowed to do.

`→` **Rule 1:**

> One: write to the database. One transaction, every table it touches. *Including a row that says "something else must happen".* Hold on to that; it's the main idea of the next 30 minutes.

`→` **Rule 2:**

> Two: respond. The caller gets their 201 as soon as the commit lands. Not when the email went out.

`→` **Rule 3:**

> Three: nothing else. No `launch { }` coroutine that sends the email in the background. No "best effort" send after commit. A background coroutine lives only in memory; when the pod dies, the work dies with it.
>
> *If it matters, it's a row. A worker makes it happen.* That's the line to remember from this slide.

**Next:** "Let's see exactly how a dual write can go wrong."

### Slide 5 · Four ways a dual write can end

**Time:** 4 min · leave at 0:15

**Clicks:** on entry, the timeline draws BEGIN → INSERT → COMMIT. `→1` pod killed; `→2` the two honest outcomes (green); `→3` the two bugs (red); `→4` the conclusion.

**Say:**

> Here's the timeline. Begin, insert the order, commit. The order is safe in the database.

`→1`

> Now the pod is killed. A deploy, out of memory, a node drain. The SQS send and the email never run.
>
> There are two things that can succeed or fail: the commit and the send. So four combinations.

`→2`

> Two of them are honest. Both succeed: the happy path. This is what every demo looks like. Both fail: nothing happened, the client gets an error and retries. Annoying, but honest.

`→3`

> The other two are bugs. Commit succeeds, send fails: a **lost event**. An order with no email, and nothing in the logs. Commit fails, send succeeds: a **ghost event**. An email for an order that doesn't exist.

`→4`

> And here's the important part. *Reordering the two calls only swaps which bug you get.* You can't fix this by being clever with the order. One system has to be the source of truth, and that's the database. The pattern is called the transactional outbox. We'll build it in a few minutes.

**Ask:** *"Which of these four have you seen in a real app? Maybe as a user?"* (Interns may have: "I paid but never got a confirmation", "I got an email for something that failed".)

**If asked "why not a distributed transaction?":** "SQS and email vendors don't support it, and where it exists it's slow and fragile. Making the database the only source of truth is simpler." (Study guide §1.)

**Next:** "Before the outbox, there's one more source of duplicates: the client."

### Slide 6 · Idempotency at the edge

**Time:** 4 min · leave at 0:19

**Clicks:** `→` ×3 reveals the three cards.

**Say:**

> New word: **idempotent**. An operation is idempotent if doing it twice has the same effect as doing it once. Pressing the elevator button five times still calls one elevator.
>
> Here's the problem. The mobile app sends `POST /orders`. Our API is slow today, so after 10 seconds the app gives up and retries. But the first request actually succeeded. It just took 11 seconds.

`→` **Card 1:**

> So the API creates a second order. And because it's a new order, it gets a new event with a new id. Later we'll see that consumers dedupe by event id. But these are two *different* ids. The consumer happily processes both. The customer is charged twice.
>
> So there are two kinds of duplicate. The queue delivering the same message twice: that's fixed in the consumer. And the user's one action arriving twice: that has to be fixed here, at the API.

`→` **Card 2:**

> The fix: the client sends an `Idempotency-Key` header. It's a random id, generated *once per user intent*. "The user tapped Pay" is one intent. Every retry of that tap reuses the same key. If the client makes a new key on every retry, it's useless.

Point at the code:

> In the API, inside the same transaction: look up the key. If we've seen it, return the stored response. If not, create the order, the outbox row and the key together. There's a unique constraint on org id and key. So if two duplicates arrive at the same moment, one commits and the other hits the constraint and returns the stored response.

`→` **Card 3:**

> We keep keys long enough to cover every client retry, usually 24 hours, then a cleanup job deletes them.

**Next:** "Let's watch a retry happen, step by step."

### Slide 7 · The retry, step by step

**Time:** 3 min · leave at 0:22

**Clicks:** `→1` first request commits; `→2` the 201 is lost; `→3` the retry with the same key; `→4` stored response replayed.

**Say:**

> Three lanes: the client, our API, Postgres.

`→1`

> The client sends `POST /orders` with key `9f1c`. The API inserts the key, the order and the outbox row in one transaction. Commit. Order 1042 exists.

`→2`

> The API sends back 201 Created, but the client already gave up at 10 seconds. The response is lost. From the client's point of view, nothing happened.

`→3`

> So the client retries, with the *same* key, `9f1c`. The API looks it up and finds it: order 1042.

`→4`

> The API returns the stored response: 201, order 1042. No second order, no second outbox row. *One intent, one order, one event, one charge.*

**Check for understanding:** *"What if the client generated a new key for the retry?"* Answer: it would look like a new intent → second order, second charge.

**Next:** "Now we know what the request writes. Let's see the whole journey of that event."

---

## Part 2 · The pipeline (0:22–0:50)

### Slide 8 · The whole pipeline

**Time:** 4 min · leave at 0:26

**Clicks:** on entry, the boxes and wires fade in. `→1` request (highlight on the three tables); `→2` relay to SQS; `→3` consumer and email; `→4` poison message to the DLQ.

**Say:**

> This is the map for the rest of the session. Every slide after this zooms into one box. Take a few seconds to look at it.
>
> In the middle band: one Postgres database with four tables: idempotency keys, orders, outbox, and processed.

`→1`

> Step one, the request. The API writes the key, the order and the outbox row in one transaction, then returns 201. The work has left the request, safely, as a row.

`→2`

> Step two, the relay. A loop running on every pod. It picks up unsent outbox rows and publishes them to the SQS queue `orders-events`.

`→3`

> Step three, the consumer. It receives the message, checks the `processed` table to see if it already did this one, does the work, commits, and acknowledges. Then the email goes out.

`→4`

> Step four, the bad case. Some messages always fail: bad data, a bug. After 5 failed attempts, SQS moves it to a dead-letter queue, `orders-dlq`, and an alarm pages someone.
>
> The title says: *one durable row in the request, then at-least-once all the way down.* Every step after the request may repeat work. None of them may lose it.

**Analogy (optional):** "A restaurant. The waiter doesn't cook. They write your order on a ticket and pin it on the kitchen rail: that's the outbox. The kitchen picks tickets off the rail: that's the relay and consumer. If a cook leaves, the ticket is still on the rail."

**Next:** "Let's zoom into the outbox."

### Slide 9 · Transactional outbox

**Time:** 4 min · leave at 0:30

**Clicks:** none; the three cards appear on entry.

**Say:**

> Three parts.
>
> **One, in the request.** One transaction: insert the order, insert an outbox row with the event type, a small payload and the org id. Both rows land, or neither does. That's the dual-write problem solved. There's only one commit now.
>
> **Two, the relay.** Claim a batch of rows, commit, publish them to SQS, then mark them sent. It runs on every pod. If a pod crashes after publishing but before marking sent, the claim expires and another pod publishes them again. So a message can be published twice. That's what at-least-once means.
>
> **Three, the consumer.** Dedupe by event id. Insert the id into `processed`. If it's already there, skip. The dedupe and the work commit together, and the ack comes after.
>
> *Three parts, three owners of correctness.* The request makes it durable. The relay makes it delivered. The consumer makes it happen once.
>
> One practical note at the bottom: delete or archive sent rows on a schedule. Otherwise the outbox becomes the biggest table in the database.

**Ask:** *"Why is it OK that the relay might publish the same row twice?"* Answer: the consumer dedupes by event id.

**Next:** "The relay looks simple, but it hides the most subtle bug of the day."

### Slide 10 · The relay

**Time:** 4 min · leave at 0:34

**Clicks:** walk the SQL first, then `→` ×3 for the cards.

**Say:** walk the SQL from top to bottom. Point as you go.

> Step one, the claim, in its own short transaction. Find rows that are not sent yet, and not claimed, or whose claim has expired. Take up to 100, in id order.
>
> `FOR UPDATE SKIP LOCKED`: lock these rows, and if another pod has already locked some, *skip* them instead of waiting. That's how ten pods run the relay at the same time without fighting.
>
> Then we set `claimed_until` to 30 seconds from now. That's a lease: "I'm working on these for 30 seconds." And then we commit. The locks are gone.
>
> Step two, publish to SQS. No locks held. Step three, mark sent, but only the ids we actually published.

`→` **Card 1:**

> Why not do it all in one transaction? Because holding a lock while you wait for the network is dangerous. If SQS is slow today, every relay pod holds a database connection and row locks the whole time. The connection pool runs out, the API starts waiting, the cleanup job is blocked. The claim column replaces the long lock. And if a pod crashes, its claim simply expires, and another pod picks up the batch.
>
> *Never hold a row lock across a network call.* That rule applies everywhere, not just here.

`→` **Card 2:**

> Order is not preserved. Ten pods publish ten batches in parallel. If order matters, put a sequence number in the payload, or use a FIFO queue.

`→` **Card 3:**

> Polling is one way to build a relay. The other is CDC, change data capture, reading the database's log with a tool like Debezium. Faster, but more to run. We poll because it's one table and one loop.

**Next:** "Let me show you what that change does to lock time."

### Slide 11 · Lock time, before and after

**Time:** 3 min · leave at 0:37 *(cut first if behind)*

**Clicks:** `→1` old relay; `→2` red lock bar; `→3` new relay; `→4` crash card.

**Say:**

`→1`

> Before: one transaction. A short select, then the publish to SQS, which is slow today, 1 second. Then a short update.

`→2`

> The red bar is how long row locks are held: 1.08 seconds. The whole slow call.

`→3`

> After: claim, commit, publish, mark sent. Two tiny lock windows, 40 and 15 milliseconds. The slow call holds nothing.

`→4`

> And if the pod crashes right after the claim? Nothing is lost. The rows stay claimed for 30 seconds, then any pod picks them up. Consumer dedupe makes the repeat harmless.
>
> The numbers are illustrative. The shape is the point.

**Next:** "The relay publishes to SQS. Let's understand the queue."

### Slide 12 · The queue

**Time:** 4 min · leave at 0:41

**Clicks:** `→` ×4 reveals the four settings.

**Say:**

> *SQS delivers at least once. Everything else follows from that.* Almost always once. Sometimes twice. Your code must be fine with twice.
>
> Look at the message. An `id`: that's what we dedupe on. A `type` and `version`, so consumers can evolve without breaking. The `org_id`, so the consumer scopes its queries to one customer. The `traceparent`, so the trace continues; we'll see that later. And the payload has *ids, not whole objects*. The consumer reads the current order from the database. A copy of the order in the message would be stale by the time it's read.

Quickly explain how SQS works, using the whiteboard if helpful:

> When a consumer receives a message, SQS doesn't delete it. It *hides* it for a while. If the consumer deletes it in time, that's the ack, and it's gone. If not, the message reappears and someone gets it again.

`→` **Visibility timeout:**

> That "while" is the visibility timeout. It has to be longer than the work. Otherwise a second consumer gets the message while the first is still working.

`→` **maxReceiveCount:**

> After this many failed attempts, the message stops looping and moves to the dead-letter queue.

`→` **DLQ:**

> The dead-letter queue is where poison messages go. We alert when it's not empty. After fixing the bug, we "redrive" the messages back to the main queue.

`→` **Long polling:**

> `WaitTimeSeconds 20`: the receive call waits up to 20 seconds for a message instead of returning empty immediately. Fewer empty calls, lower cost, faster pickup.

**Next:** "Let's see the first two settings on a timeline."

### Slide 13 · Visibility timeout and the DLQ

**Time:** 4 min · leave at 0:45

**Clicks:** `→1` consumer A starts; `→2` message reappears, B takes it; `→3` the fix; `→4` five receives to the DLQ.

**Say:**

`→1`

> Consumer A receives the message. It's hidden for 30 seconds. But today the work takes 45 seconds: maybe the email vendor is slow.

`→2`

> At 30 seconds, the message is visible again. Consumer B takes it. A finishes and sends the email. B also sends the email. Without dedupe: *two emails*.

**Ask:** *"How would you fix this?"* Expect: "make the timeout longer" and "dedupe". Both are right.

`→3`

> Set the visibility timeout above the p99 handling time, and dedupe anyway. Duplicates still happen during deploys and pauses.

(p99: "99% of messages are handled faster than this.")

`→4`

> And a message that always fails, maybe bad data. Receive 1, fail. 2, 3, 4, 5, fail. maxReceiveCount is 5, so it moves to the DLQ and the alarm fires. Without a DLQ, it would loop forever.

**Next:** "Now the consumer itself."

### Slide 14 · The consumer

**Time:** 3 min · leave at 0:48

**Clicks:** `→` ×4 reveals the four cards.

**Say:** walk the code.

> Receive up to 10 messages. For each one: open a transaction. Try to insert the message id into `processed`. If it's already there, we've done this one: skip. Otherwise do the work. Commit. *Then* delete the message from SQS, the ack. If anything throws, log it with the exception, and *don't* delete. SQS will redeliver it later.

`→` **Card 1:**

> The try/catch is *per message*. One bad message must not stop the other nine.

`→` **Card 2:**

> Keep the transaction small. No outside calls inside it, the same rule as the relay. If you need to call a vendor, commit first and then call, or write another outbox row.

`→` **Card 3:**

> Ten pods run this loop at the same time. It's correct because of the dedupe table, not luck.

`→` **Card 4:**

> The consumer gets its own thread pool and its own database connections. If it shares the API's pool, a big queue backlog starves the API.

**Next:** "Why commit, then ack? Let's crash it."

### Slide 15 · Where can it crash?

**Time:** 2 min · leave at 0:50

**Clicks:** `→1` ack first + crash; `→2` the right order; `→3` crash before the ack; `→4` the rule.

**Say:**

`→1`

> The wrong way: receive, delete the message, then crash before the work. The message is gone from SQS. The work never happened. *Lost for good.* No redelivery, no DLQ, no log line.

`→2`

> The right way: receive, open a transaction, dedupe, do the work, commit.

`→3`

> Now crash before the ack. SQS redelivers after 30 seconds. The dedupe finds the id, skips the work, acks. Work done once, message gone.

`→4`

> *Only ack-first can lose work.* Commit-then-ack can only repeat it, and dedupe makes the repeat harmless.

**Ask:** *"What if it crashes before the commit?"* Answer: the transaction rolls back, nothing happened, the message runs again. Fine.

**Next:** "Let's check that this all makes sense before the break."

---

## Exercise 1 + break (0:50–0:57)

**Exercise 1 (3 min): Draw the pipeline.** See `03-qa-and-exercises.md`, Exercise 1. Ask interns, in pairs, to draw the pipeline from memory on paper: client → API → tables → relay → SQS → consumer → DLQ, and write one sentence per box about what can go wrong. Show slide 8 again to check.

**Break (4–5 min):**

> Five-minute break. When we come back: what happens when something we depend on is slow.

While they're on break, glance at the clock and decide whether you need to cut anything from Part 3.

---

## Part 3 · Slow dependencies (0:57–1:21)

### Slide 16 · When a dependency is slow

**Time:** 4 min · leave at 1:01

**Clicks:** cards appear on entry. `→` reveals Little's law.

**Say:**

> Welcome back. Question first.

**Ask:** *"Which is worse: a dependency that's down, or one that's slow?"* Let them vote. Most say "down".

> Slow is worse. When a service is down, the connection fails in a millisecond and you move on. When it's slow, every request waits. Your threads wait, your database connections wait, new requests queue behind them. Soon your whole service is slow, even endpoints that never call that vendor.
>
> Four tools, one goal: *fail fast and alone.* Fail quickly, and fail only the part that depends on the broken thing.
>
> **Timeouts at every hop, each shorter than its caller's.** The client waits 10 seconds. The API gives the vendor 3. The vendor call gives its connection 1. If an inner timeout is longer than the outer one, you're still working after the caller has left.
>
> **Retry with backoff, jitter and a budget.** Three attempts. Wait longer each time. Add randomness. Only retry errors that can succeed next time: timeouts, 503s. Never a 400: a bad request is still bad the second time. And only retry calls that are safe to repeat.
>
> **Circuit breaker.** After enough failures, stop calling for a while. Fail immediately. Then let one request through to test.
>
> **Bulkhead.** The vendor gets its own small thread pool. When it hangs, only that pool fills up.

`→` **Little's law:**

> How big should a pool be? Little's law: *requests in flight equals arrival rate times time per request.* 50 requests per second at 200 milliseconds each: 50 × 0.2 = 10 in flight. If the vendor slows to 3 seconds: 50 × 3 = 150 in flight. A pool of 20 can't hold that. That's why the vendor gets a bulkhead.

Do the arithmetic out loud, slowly.

**Next:** "Let's draw the timeouts and the breaker."

### Slide 17 · Timeouts and breakers, drawn

**Time:** 4 min · leave at 1:05

**Clicks:** `→1` nested timeouts; `→2` the bad case; `→3` the circuit breaker.

**Say:**

`→1`

> Client 10 seconds. API to vendor, 3. Connect, 1. Each one fits inside its caller. Like Russian dolls.

`→2`

> The bad case: the vendor call has a 15-second timeout. At 10 seconds the client gives up. We keep working for 5 more seconds, for nobody. We pay the cost and nobody gets the result.

`→3`

> The circuit breaker is a small state machine. Watch the colors cycle.
>
> **Closed**, green: calls flow normally. We count failures. After N failures in a window, it opens.
>
> **Open**, red: we don't even try. We fail immediately and return a fallback. After a cooldown, it goes half-open.
>
> **Half-open**, orange: we let one probe request through. If it succeeds, closed again. If it fails, back to open.
>
> The note at the bottom matters: *while the breaker is open, return the fallback right away.* A breaker that still waits for a timeout isn't saving anything.

**Analogy:** "It's the fuse box in your house. When something shorts, the fuse trips so the whole house doesn't burn. Later you flip it back and see if it holds."

**Next:** "And bulkheads, visualized."

### Slide 18 · Bulkheads, visualized

**Time:** 5 min · leave at 1:10

**Clicks:** `→1` shared pool; `→2` split pools; `→3` Little's law calculator.

**Say:**

`→1`

> One shared pool of 20 threads. The vendor hangs. One by one, every thread is stuck waiting on it. Now `GET /health`, `GET /me`, `POST /orders` are all waiting for a thread. The health check times out, and Kubernetes restarts a *healthy* pod. Now we have fewer pods and the same problem.
>
> The name "bulkhead" comes from ships: the hull is split into compartments so one leak doesn't sink the whole ship.

`→2`

> A pool per dependency: 5 for the vendor, 15 for the API. The vendor's 5 fill up. More vendor calls fail fast with a 503. But health answers in 12 milliseconds, and everything else keeps serving.

`→3` **Calculator.** Drag the sliders live:

> Let's try it. 50 requests per second, 200 milliseconds: 10 in flight. Fits easily in a pool of 20; the white mark is 20. Now the vendor gets slow: 3000 milliseconds. 150. Way over.

**Ask:** *"Someone give me a request rate and a latency."* Let an intern pick numbers. Ask the room to calculate before you move the slider.

(Note: the sliders don't change slides. Click outside them, or use the keyboard, to move on.)

**Optional, if ahead:** Exercise 3 in `03-qa-and-exercises.md`.

**Next:** "Some fine print about retries."

### Slide 19 · The fine print

**Time:** 3 min · leave at 1:13

**Clicks:** the fan-out tree grows on entry. `→1` deadlines; `→2` degradation.

**Say:**

> Retry amplification. Three layers: say the mobile app, our API, and an internal service. Each retries 3 times. One click becomes 3, then 9, then *27 calls* on a vendor that is already struggling. We're attacking our own vendor.
>
> Fix: *retry in one place*, usually the layer closest to the dependency.

`→1`

> Propagate the deadline, not a fixed timeout. If the client gave us 10 seconds and 7 are gone, the vendor call gets what's left, not its usual 3. Pass the deadline down, and check it before every retry.

`→2`

> And decide what "failing fast" returns. An error is the last option. Better: a cached answer, a smaller feature, or "accepted, we'll finish later": a 202 and an outbox row. That's the outbox again.

**Next:** "One more detail about retries: timing."

### Slide 20 · Backoff and jitter

**Time:** 3 min · leave at 1:16 *(shorten if behind)*

**Clicks:** `→1` lockstep; `→2` full jitter; `→3` capacity line and the numbers.

**Say:**

> Imagine 200 clients all fail at the same moment, because the vendor had a blip.

`→1`

> Without jitter, they all wait exactly 200 milliseconds, and retry at the same moment. Then all wait 400 more and retry together again. 200 requests at once, every time. Each wave knocks the vendor over again. This is called a *thundering herd*.

`→2`

> With full jitter, each client waits a *random* time between zero and the backoff. The same retries, spread out.

`→3`

> The orange line is the vendor's capacity. Red bars are over it. Same 600 retries in total, but the peak drops a lot. (It's a small simulation computed in the page; the exact number changes on each load.)

**Next:** "What about when we are the one being overloaded?"

### Slide 21 · Backpressure

**Time:** 3 min · leave at 1:19

**Clicks:** `→1` unbounded curve; `→2` the knee; `→3` bounded with 429s.

**Say:**

`→1`

> This is p99 latency as load increases. Flat, flat, flat, then it shoots up. That point is called the knee.

`→2`

> Here at 240 requests per second, the database pool of 20 is full. Every request after that waits in a queue, and the queue keeps growing. A queue with no limit grows until the pod runs out of memory, and then everything in it is lost at once.

`→3`

> With a bound: when we're full, we reject immediately with a 429 or 503 and a `Retry-After` header. The rejected requests fail in milliseconds. Everyone else stays fast. *A fast no beats a slow yes.*
>
> How do you find your knee? Load-test with a tool like k6: ramp up traffic, watch where latency jumps, and find which limit caused it. That number goes in the runbook.

**Analogy:** "A popular restaurant that says 'full, come back at 8' is better than one that seats everyone and serves dinner at midnight."

**Next:** "Last topic in this part: leaving cleanly."

### Slide 22 · Graceful shutdown

**Time:** 2 min · leave at 1:21

**Clicks:** `→1` SIGTERM and readiness; `→2` each loop drains; `→3` exit and the 30-second line.

**Say:**

> *Every deploy is a shutdown.* If shutdown isn't clean, every deploy drops some requests.

`→1`

> Kubernetes tells the pod to stop. A short `preStop` sleep of 5 seconds gives the load balancers time to stop sending us new traffic. Then the app gets SIGTERM.

`→2`

> Then each loop finishes its job. The HTTP server finishes the requests in progress. The consumer stops receiving and finishes its current batch. The relay stops claiming and publishes what it already claimed.

`→3`

> Close the database pool, exit 0. At 9.5 seconds, we're done. Kubernetes would send SIGKILL at 30 seconds. Anything still running then is cut off. But thanks to dedupe and claims, a hard kill delays work, it doesn't lose it.

**If asked about the exact order:** the preStop hook runs first, then SIGTERM is sent; the 30 s counts from the start. (Study guide §11.)

**Next:** "Now: how do we *know* any of this is happening?"

---

## Part 4 · Observability (1:21–1:29)

### Slide 23 · Observability

**Time:** 5 min · leave at 1:26

**Clicks:** cards appear on entry. `→1` page on symptoms; `→2` runbooks.

**Say:**

> Three kinds of signal, three questions.
>
> **Logs** answer: what happened to *this* request? Structured JSON, one event per line. Every line has the request id, org id and user id, added automatically from the MDC: think of it as a per-request "sticker" that the logger puts on every line. Errors log the exception with its stack trace. No passwords, no tokens, no personal data you don't need.
>
> **Metrics** answer: is the service healthy right now? Rate, errors, duration, called RED, per endpoint and per consumer. Use percentiles like p99, not averages: an average can hide the 1% of users who wait 10 seconds. Plus saturation: how full the pools are, how old the oldest queue message is, how big the DLQ is, and *outbox lag*.
>
> **Traces** answer: where did the time go? One trace id follows the request through the API, the database, the vendor, and the consumer. That works because we put the trace context in the HTTP headers *and in the message*.

`→1`

> What should wake someone up at night? *Symptoms, not causes.* Error rate over 1% for 5 minutes. p99 over our target. DLQ not empty. Queue age over 10 minutes. And *the oldest unsent outbox row over 5 minutes.* Not "CPU is at 80%". High CPU might be fine. Users hurting is never fine.

Pause on the outbox line: *"Remember this one. It comes back in a few minutes."*

`→2`

> Every alert needs a runbook and a person. An alert nobody acts on gets muted within a month. Fewer alerts, each one real.

**Next:** "Here's what a trace looks like."

### Slide 24 · One trace, end to end

**Time:** 3 min · leave at 1:29

**Clicks:** `→1` the request; `→2` the async hop; `→3` the consumer; `→4` why it's one trace.

**Say:**

> Where did 4.2 seconds go?

`→1`

> The API request: 180 milliseconds, with two short database spans. The user got their 201 here. Fast.

`→2`

> Then the event waits about a second for the relay's next poll. That's the cost of polling. Then the publish, 60 milliseconds, and 30 milliseconds in the queue.

`→3`

> Then the consumer: 2.9 seconds. Almost all of it is the email vendor, 2.83 seconds. *That's where the time went.*

`→4`

> And it's one trace, from the POST to the email, because `traceparent` rides along in the SQS message. Without it, we'd see two unrelated traces and have to guess.

**Next:** "Enough slides. Let's look at the real thing."

---

## Live walkthrough (1:29–1:43)

### Slide 25 · Live in the repo

**Time:** 14 min · leave at 1:43

Leave this slide on screen as the agenda, then switch to your tabs. The deck plans 30 minutes for this; for interns, 14 minutes on the key steps is enough. Fill in the paths before the session:

| # | Step | Open | Point out | Minutes |
| --- | --- | --- | --- | --- |
| 1 | The outbox write | `[path to the order handler / service]` | The outbox insert is inside the same transaction as the order. The idempotency key check. | 2 |
| 2 | The relay | `[path to the relay]` | Claim, commit, publish, mark sent. Batch size. What happens on a publish failure. | 2 |
| 3 | The consumer | `[path to the consumer]` | Dedupe, transaction, ack order. Its own executor and pool size. | 2 |
| 4 | Queue config | `[path to the Terraform for the queue]` | Visibility timeout, `maxReceiveCount`, the DLQ and its alarm. | 2 |
| 5 | Dashboards | `[dashboard link]` | RED per endpoint, queue age, DLQ, pool saturation, outbox lag. The alert rules. Follow one real trace id from the POST to the consumer. | 3 |
| 6 | The incident | `[incident dashboard / graph]` | Back to the 3 a.m. graph. Let the room drive. | 3 |

**For each code file, ask the room one question before you explain:**

- Outbox write: *"Find the transaction. Is the outbox insert inside it?"*
- Relay: *"Where does the transaction end? Is any network call inside it?"*
- Consumer: *"Which line is the ack? Is it before or after the commit?"*
- Terraform: *"Is the visibility timeout longer than how long the consumer takes?"*

**Step 6, let the audience drive:**

> Back to 3:07 a.m. You have the dashboards now. Where do you look first? … And then? … What would have caught this earlier?

Let them suggest. Gently guide them to the outbox table / outbox lag. Then go back to the deck, slide 26.

**If you can't show a real repo** (no access, interns can't see the code, or the demo breaks): do Exercise 4 in `03-qa-and-exercises.md` instead. It's a code-reading exercise on paper using the snippets from the slides. Same lessons, no laptop risk.

---

## The answer, recap, review (1:43–1:50)

### Slide 26 · 03:07 a.m., answered

**Time:** 3 min · leave at 1:46

**Clicks:** on entry, the same graph as slide 3. `→1` alert at 02:15; `→2` customers noticed at 03:07; `→3` the takeaway.

**Say:**

> Same graph as the beginning.

**Ask:** *"Now that you've seen the whole pipeline: what happened?"* Wait. You want: "The relay stopped. Nothing was being published, so the queue was empty, and an empty queue looks healthy."

Point back at the guesses on the whiteboard from slide 3, if you wrote them. "Which of these would have found it?"

> Exactly. The relay stopped at the deploy. The queue was empty because *nothing was being published*. An empty queue looks exactly like a healthy queue. So every queue alert stayed quiet.

`→1`

> With an outbox-lag alert, "the oldest unsent row is older than 5 minutes", someone gets paged at 2:15.

`→2`

> Customers noticed at 3:07. That's 52 minutes earlier.

`→3`

> The fix was one query and one alert rule. But the lesson is bigger: *every async hop needs its own "is work piling up here?" signal.* Outbox lag for the relay. Queue age for the consumer. DLQ size for poison messages.

(If you used a real incident on slide 3, tell its real ending here.)

**Next:** "Let's recap with numbers."

### Slide 27 · The numbers to remember

**Time:** 2 min · leave at 1:48

**Say:** go quickly; each number is a reminder, not a new idea. Point and say one line each:

> 27 calls: one click with retries at three layers. Retry in one place.
> 150 in flight: 50 requests a second at 3 seconds. Why the vendor gets its own pool.
> 10, 3, 1 seconds: each timeout shorter than its caller.
> 30 seconds: relay claim expiry.
> 5 receives: then the DLQ.
> 24 hours: how long we keep idempotency keys.
> 1% for 5 minutes: the error rate that pages.
> 10 minutes: queue age that pages.
> 5 minutes: oldest unsent outbox row that pages.
>
> These are examples. Your service's real numbers may be different; the reasoning is the same.

**Next:** "And these are the mistakes you'll see in code review."

### Slide 28 · Ten reliability mistakes review sees most

**Time:** 2 min · leave at 1:50

**Say:**

> This is the review checklist. When you open a PR that touches a queue, a vendor call, or background work, read this list first.

Go through them fast; they're all from earlier slides. Then:

**Ask:** *"Hands up: which of these do you think you'd be most likely to write?"* Usually `launch { }` and "no timeout" get the most hands. That's normal; say so.

> Don't worry about remembering all ten. Bookmark this slide. Review will catch you, and that's how everyone learns these.

**Next:** "Questions."

---

## Q&A and close (1:50–2:00)

### Slide 29 · Close

**Time:** 10 min · leave at 2:00

**Say:**

> Four sentences to take home. *If it matters, it's a row. Dedupe, commit, ack. Fail fast and alone. Page on symptoms.*
>
> If you want the long version: the book *Release It!* by Michael Nygard, and the transactional outbox page on microservices.io.
>
> And the line at the bottom: *ask anyone, anything. Don't hide and guess.* It's true for this session and for your whole internship. Now, questions. Every intern asks at least one.

**If questions are slow to start:**

1. *"Which of the ten mistakes would bite our service first?"*
2. *"What would happen in our system if the email vendor was down for an hour?"* (Answer: the breaker opens, messages fail and are retried, some may reach the DLQ; nothing is lost because the work is in the queue or DLQ; redrive after the vendor recovers.)
3. *"If you were on call tonight, which dashboard would you open first?"*

See `03-qa-and-exercises.md` for prepared answers to the questions interns ask most.

**Close:**

> Thanks, everyone. The deck is at [link]. If a question comes to you tonight, send it to me on Slack. That counts too.

Optionally share the quiz in `03-qa-and-exercises.md` as homework.
