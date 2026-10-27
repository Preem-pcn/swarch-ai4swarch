> Reviewer agent: Claude Code (Anthropic), model Claude Opus 5.5. Reviewed: `docs/adr/0002-publish-order-events-via-rabbitmq.md`, against `docs/scenario.md` only.

# Architecture review (ATAM style): ADR-0002, OrderPlaced events via RabbitMQ

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 Performance | Customers | 200 `PlaceOrder`/s | 11.11 peak; MailFast at its p99 of 8 s | Order confirmed; `OrderPlaced` stored durably; no MailFast call on the request path | p95 `PlaceOrder` ≤ 500 ms (today ≈ 300 ms) |
| S2 Availability | Infrastructure fault | RabbitMQ, relay, `loyaltyservice` or MailFast down for several minutes | Peak load | Orders still succeed; events wait and are delivered after recovery | 100 % order success for paid and shipped orders; 0 lost events |
| S3 Correctness / latency | `checkoutservice` | Order placed, then the relay or a consumer crashes and the message is redelivered | 200 orders/s | Points awarded once | p99 order-to-points ≤ 60 s; 0 duplicate awards |

S2 and S3 conflict. If RabbitMQ or the relay is down for more than 60 s, S2 is met only by breaking S3. The ADR does not say which one wins.

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | The "transactional" outbox has no local transaction to be atomic with. `checkoutservice` has no store (ADR, Context table), and `chargeCard` and `shipOrder` are remote gRPC calls. The outbox row is written *after* `shipOrder` (Design rules). | A crash between `chargeCard` and the outbox write still loses the event for a charged customer. The ADR says this loss is solved (Risks, "Mitigation (decided)"). It is only made smaller. |
| Risk | The new order store sits synchronously on the order path, and its technology is "Not decided". | The ADR does not say what happens when the store is down: either the order fails (a new availability dependency) or the event is lost. It is missing from the failure-injection list and from the 200 ms latency budget. |
| Risk | The outbox relay is a single funnel for every consumer, and its HA and poll interval are not specified. | A relay that is down or lagging delays loyalty for everyone (S3). Running several relay replicas without locking adds more duplicates. |
| Sensitivity point | Email backlog = (200 − 50) msg/s × peak duration. It drains only while the order rate stays below 50/s. | Example **[Inferred]**: a 30-min peak leaves 270,000 queued emails. The last peak customer waits about 90 min. If post-peak traffic is ≥ 50 orders/s, the backlog never drains. |
| Sensitivity point | Loyalty lag = relay poll interval + broker + consumer time at ≥ 200 msg/s (not benchmarked). | The 60 s target depends on numbers the ADR does not set. |
| Trade-off point | Outbox: reliability (fewer lost events) versus latency, availability and operability (DB write on the hot path, a new store, a relay). | The ADR records only the operability cost (Consequences). |
| Trade-off point | Async email: checkout latency and availability versus how quickly customers get their confirmation. | The ADR handles this correctly by requiring business sign-off (Consequences). |
| Non-risk | One durable queue per consumer, a token bucket at ≤ 50/s, and a DLQ for the ≈ 0.5 % failures. | Isolates MailFast slowness and outages from loyalty and analytics. Retries add only about 0.25 msg/s. |
| Non-risk | Dedup on `order_id` inside the loyalty points transaction; the relay marks a row sent only after a publisher confirm. | Gives exactly-once *effect* on top of at-least-once delivery, which is the correct reading of the requirement. |
| Non-risk | Option 2 rejected; fraud scoring scoped out because it may need to run before the order is accepted. | In-memory work is lost on pod restart. Fraud scoring is a different, synchronous concern. |

## 3. Missing or weak

- **Options not considered:**
  - Writing a *pending order* record before `chargeCard` and updating it afterwards, so a crash leaves a record that can be reconciled. This is the only way to actually close the dual-write gap here.
  - Change data capture versus a polling relay.
  - RabbitMQ Streams for analytics replay, which may make the "reconsider Kafka" trigger unnecessary **[Inferred]**.
- **Consequences not stated:**
  - Event schema and versioning for future consumers (the modifiability driver). For example, protobuf, which fits the team's gRPC skills.
  - Behaviour when the order store is down.
  - Which wins when the loyalty 60 s target and RabbitMQ downtime conflict.
- **Unsupported or likely wrong claims:**
  - "The only option that meets drivers 1-5" (Decision, "Why"). Option 4 also meets drivers 1-5; only driver 6 separates them.
  - "`PlaceOrder` adds only one in-cluster publish" (Consequences). Under the decided design, checkout writes to a DB and does not publish.
  - The Kafka con "rate limiting … must be built by hand" does not separate the options. The 50/s token bucket lives in `emailservice` under option 3 as well.
  - "A pod crash … no longer loses it" (Risks). This is false for crashes before the outbox write.

## 4. Verdict: **Accept with changes**

1. **Make the outbox real or rename it.** Write a pending order and outbox record *before* `chargeCard`, choose the store technology, and state its latency cost and failure behaviour. Add the order store and the relay to the failure-injection test.
2. **Set the loyalty budget.** Give the relay poll interval, the relay HA design and a benchmark target of ≥ 200 msg/s. State explicitly that order availability wins over the 60 s target during a broker outage, and alert on breaches.
3. **Fix the email and backlog claims.** Get a peak duration and a business-approved email delivery target, and size the queue disk from them. Correct the three inaccurate claims above.
