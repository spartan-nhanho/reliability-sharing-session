# Reliability: queues, outbox, observability

A 2-hour internal sharing session on how work leaves a request without getting lost, how a service behaves when a dependency is slow, and how you find out what happened at 3 a.m.

The main deck tells the story of one small online store. It starts from a naive design, breaks it nine times, and adds one piece to the design for each fix.

The session is delivered as a single self-contained HTML slide deck, so there is no build step and no dependencies beyond a web browser.

## Contents

| File | Purpose |
| --- | --- |
| `Reliability Engineering Session Slides - Story Mode.html` | **The main deck** (49 slides, one file, no external JS). Published to GitHub Pages. |
| `Reliability Engineering Session Slides.html` | The original deck (30 slides), updated for interns. The main deck reuses slides from an earlier version of it. |
| `README.md` | This file. |
| `docs/01-presenter-study-guide.md` | Background reading for the presenter: the long version of every topic. |
| `docs/02-presentation-script.md` | English slide-by-slide script for the original 29-slide deck. |
| `docs/03-qa-and-exercises.md` | Likely questions with answers, in-session exercises, a quiz and a glossary. |
| `docs/04-script-tieng-viet.md` | Vietnamese slide-by-slide script for the Story Mode deck (49 slides, 2 hours). |
| `.github/workflows/deploy.yml` | Publishes the deck to GitHub Pages on every push to `main`. |

## Session outline

| Part | Length | Topics |
| --- | --- | --- |
| The shop, version 0 | 10 min | The business, the naive design, the 3 a.m. mystery |
| Nine problems, one at a time | 90 min, with a break | Lost email, double charge, relay locks, duplicate delivery, lost work and poison messages, slow vendor, traffic spike, deploys, silent failure |
| End to end, the real repo, Q&A | 20 min | One order through the final design, live walkthrough, the numbers, the review checklist |

### Topics covered

The main deck covers all of these, except the parts marked *original deck only*.

- **Six words for today.** At-least-once, idempotent, ack, outbox, visibility timeout and dead-letter queue, each with an everyday analogy. *Original deck only.*
- **Dual writes.** Why writing to the database and then sending to a queue or email can lose or duplicate events.
- **Transactional outbox.** Writing the event as a row in the same transaction, then relaying it later.
- **Idempotency keys.** Deduplicating the client's retried request, not only the message; concurrent duplicates and a reused key with a different body (`422`, *original deck only*).
- **The relay.** Claim, commit, publish, mark sent. Never hold a row lock across a network call.
- **Queues (SQS).** At-least-once delivery, visibility timeout, `maxReceiveCount`, dead-letter queues, long polling.
- **The consumer.** Dedupe, do the work, commit, then ack, in that order. Side effects such as email are rows (`email_outbox`) sent by their own worker (*original deck only*).
- **Slow dependencies.** Nested timeouts, retries with backoff and jitter, circuit breakers, bulkheads, and Little's law for sizing pools.
- **Backpressure.** Bounded queues and fast `429` responses instead of unbounded latency.
- **Graceful shutdown.** What each loop does on `SIGTERM` before the pod is killed.
- **Observability.** Structured logs, RED and saturation metrics, end-to-end traces, and alerts that page on symptoms.
- **Review checklist.** Ten reliability mistakes that code review catches most often.

## Viewing the deck

1. Open `Reliability Engineering Session Slides - Story Mode.html` in a modern browser (Chrome, Safari, or Firefox).
2. Navigate with the keyboard or the mouse:

| Key | Action |
| --- | --- |
| `→` / `←` | Next / previous slide, or next / previous build step |
| Click a red marker or a "dive in" card | Jump into that problem |
| Click a green marker or a number 1–9 (top right of a map) | Revisit a problem |
| `M` | Back to the system map for the current problem |
| `N` | Show speaker notes |
| `F` | Fullscreen |

Diagram slides build step by step. The dots in the bottom-right corner show how many reveals are left on the current slide.

To print or save as PDF, use the browser's print dialog. Every slide is fully built in print mode.

The page loads one web font from Google Fonts. Without a network connection it falls back to system fonts, and the layout still works.

## Deployment

The deck is published to GitHub Pages. The workflow in `.github/workflows/deploy.yml` runs on every push to `main`:

1. Checks out the repository.
2. Copies the main deck (`Reliability Engineering Session Slides - Story Mode.html`) to `_site/index.html`, so the URL is clean.
3. Uploads `_site` as a Pages artifact and deploys it.

Pushing to `main` updates the live site automatically. You can also trigger a deploy by hand from the **Actions** tab (**Run workflow**).

One-time setup is required in the repository settings:

1. Go to **Settings → Pages**.
2. Under **Build and deployment → Source**, select **GitHub Actions**.

After the first successful run, the site is available at the URL shown in **Settings → Pages**, normally `https://spartan-nhanho.github.io/reliability-sharing-session/`.

> Note: on a private repository, GitHub Pages requires a paid GitHub plan. If the repository is private and the plan does not include Pages, make the repository public or use another static host.

## Editing the deck

The deck is one HTML file with its CSS and JavaScript inline. To change a slide, edit the matching `<section>` in the HTML file and commit to `main`. The site redeploys automatically.

Content edits do not require any tooling; the slide text is plain HTML.

## References

The deck points to *Release It!* (Michael Nygard) and the microservices.io transactional outbox page for the long version of the material.
