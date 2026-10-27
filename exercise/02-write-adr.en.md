**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No. It acknowledges that latency targets, order volume, and MailFast's rate limit are unknown. | Yes. It uses the 500 ms checkout target, current ~300 ms latency, 200 orders/s peak, MailFast's 50 emails/s limit and latency, and the one-minute loyalty requirement. |
| Options considered (list them) | Three: synchronous calls, message broker with pub/sub, and a dedicated post-order orchestrator. | Four: synchronous gRPC calls, in-process asynchronous goroutines, RabbitMQ, and Kafka. |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | **Exactly once:** recommends idempotent consumers keyed by order ID, but doesn't specify the scenario's one-minute deadline or deeply define transactional deduplication. **Rate limit:** recommends retries with backoff, a dead-letter queue and alerts, but doesn't account for the 50/s cap or the 200 orders/s peak. | **Exactly once:** explains that delivery is at-least-once, while exactly-once points awarding is achieved through deduplication by `order_id` in the same transaction. It also calls out the one-minute target and dual-write risk. **Rate limit:** explicitly identifies the 4:1 mismatch, recommends buffering in a durable queue and limiting MailFast calls to 50/s, with shared rate limiting for retries and backoff on HTTP 429. |
| Anything invented (a number, tool or fact not in the scenario)? | Assumes an existing synchronous email call and no existing broker, while acknowledging these aren't evidenced by the repository files available to the run. It also assumes the order of decision drivers. | Uses source code and scenario details, but still makes some assumptions: for example, HTTPS for MailFast, a three-node RabbitMQ cluster, and proposed service/database technologies that aren't specified by the scenario. It labels several inferences and leaves unresolved questions. |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Added a transactional outbox (new order store + outbox relay) instead of publishing directly to RabbitMQ; updated the Decision, design rules, diagram, risks and open questions. | Losing the event after charging the customer could leave the order without a confirmation email or loyalty points. The event should be persisted reliably before asynchronous publishing. |
| 2 | Documented the email delay as needing business approval and a delivery-time target. | At peak load, the system may accumulate around 9,000 emails per minute beyond MailFast's sending capacity. The business should approve the expected delay and define a delivery-time target. |
| 3 | Kept RabbitMQ, and added when to reconsider Kafka. | The team already has experience with RabbitMQ and it is sufficient for the current post-order messaging requirements. Kafka should be reconsidered if future requirements make event replay, long-term retention, or stream processing important. |
| 4 | Moved fraud scoring out of scope (removed it from "Future actions"). | Fraud decisions may need to happen before an order is accepted, unlike asynchronous post-order actions. It should have a separate architecture decision. |
| 5 | Replaced the AI attribution with my name and added myself under Deciders. | I am responsible for reviewing and approving the final ADR. |
