# Worksheet 04 — Architecture Review Board


## ADR under review

- **ADR:** ADR-0002 — Publish an `OrderPlaced` event to RabbitMQ for post-order actions
- **Author:** Pitchanan Thaiphayak
- **Decision under review:** Use self-hosted RabbitMQ, an outbox, and independent consumers for post-order actions.

## Board members and roles

| Role | Name |
|---|---|
| Architect (ADR author) | Pitchanan |
| AI Analyst | Kotchapan |
| Product Owner (PO) | Pon |
| Site Reliability Engineer (SRE) | Piyapanchanok |
| Security Officer | Chayutpong |


### 1. Architect — Introduces the ADR

The architect explains that RabbitMQ decouples post-order actions from checkout. Email, loyalty, and analytics can be processed independently, and future consumers can be added without tightly coupling them to the order service.

The architect acknowledges that the design still needs to clarify what happens if payment succeeds but the order event is not durably recorded. The architect also asks the board to consider whether the proposed design meets the loyalty latency target during broker outages.

### 2. AI Analyst — Presents the AI review

The AI Analyst presents the review findings without adding personal opinions:

- The payment-to-outbox boundary may allow a successful payment to occur without a durable event.
- The design needs clearer relay behavior, recovery guarantees, and handling of loyalty processing delays.
- At peak load, email demand can exceed MailFast's throughput, creating a growing backlog.
- Some claims comparing RabbitMQ with Kafka are not adequately supported.
- The 30-minute peak example is an illustrative assumption, not a stated scenario requirement.

### 3. Product Owner — Challenges business assumptions

The PO asks whether customers and the business can tolerate delayed confirmation emails when order volume exceeds MailFast's limit. At 200 orders per second and a maximum email throughput of 50 per second, the backlog could grow by approximately 9,000 emails per minute if those rates persist.

The PO also questions how the one-minute loyalty requirement should be handled during a broker outage. The board needs an explicit business decision about acceptable delays and recovery expectations rather than assuming that asynchronous processing automatically meets the targets.

### 4. SRE — Challenges reliability and operations

The SRE asks how the outbox relay will run, how failures will be retried, and how duplicate messages will be handled. The SRE recommends documenting monitoring, alerting, queue-depth thresholds, dead-letter handling, and recovery procedures.

The SRE also notes that the team must validate throughput and availability under realistic conditions. The ADR should not assume that the proposed broker and relay automatically satisfy the peak workload or loyalty latency target.

### 5. Security Officer — Reviews security and privacy

The Security Officer points out that queued events may contain customer email addresses and other order-related data. The ADR identifies personal data in queued events as an open question but does not yet define the necessary privacy and security controls.

The Security Officer asks the team to specify which fields are required in events, who can publish and consume messages, how broker connections are authenticated and protected, and how long messages and dead-letter records are retained. Sensitive data should not remain in queues longer than necessary, and access to messages and operational logs should be restricted.

### 6. Architect — Responds and summarizes changes

The architect agrees that the ADR needs clearer failure semantics, explicit business acceptance criteria, and stronger security requirements. The architect proposes revising the ADR before implementation rather than treating the existing design as fully validated.

##  decision

**Decision: Accept with changes.**

** vote:** 5–0 in favor of accepting the ADR subject to the required changes below. Replace this vote with the actual vote from your meeting.

## Required changes

1. **Clarify payment and outbox consistency.** Document the transaction boundaries between payment, order persistence, and event publication. Define recovery and reconciliation behavior if a failure occurs between these steps.
2. **Specify delivery and recovery guarantees.** Document relay operation, retries, duplicate handling, dead-letter processing, monitoring, and recovery. Validate that the system can meet its workload and latency targets.
3. **Define business-level service expectations.** Obtain agreement on acceptable email delays, backlog recovery, and the one-minute loyalty requirement during broker outages. Support broker comparisons with evidence rather than unsupported claims.
4. **Add security and privacy requirements.** Minimize personal data in events; define broker authentication, authorization, connection protection, retention, and dead-letter queue access.
5. **Validate the design before production.** Test peak throughput, broker outages, consumer failures, duplicate delivery, and recovery behavior.

## Concerns raised

| Concern | Raised by  | Covered by AI review? | Could AI have known? |
|---|---|---|---|
| Payment may succeed without a durable event | Architect and SRE | Yes | AI can infer the consistency risk from the documented transaction boundary; the correct recovery policy needs human agreement. |
| Loyalty may miss its one-minute target during an outage | PO and SRE | Yes | AI can identify the conflict; business priorities and acceptable degradation require stakeholder decisions. |
| Email backlog may grow at peak load | PO | Yes | AI can calculate the backlog rate; acceptable delay and peak duration need confirmation. |
| Broker comparison claims may be unsupported | Architect | Yes | AI can flag unsupported statements and request evidence. |
| Personal data may be exposed or retained in queues and dead-letter records | Security Officer | Not identified as a main finding in the review summary; the ADR does list privacy as an open question | AI could flag the open question and suggest controls, but the organization must define its security and privacy requirements. |

## Best AI finding

The payment-to-outbox consistency gap was a strong finding because it exposed a failure scenario that the architecture needs to address explicitly. It helped the board distinguish between having an outbox and guaranteeing that the payment and event workflows remain consistent.

## Weakest AI finding

The 30-minute peak example could be misleading if interpreted as a scenario requirement. The backlog calculation is useful, but the peak duration is an assumption and should be labeled as such.