# ADR-0002: Use RabbitMQ for asynchronous post-order event processing

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Online Boutique Engineering Team

## Context and problem

In Online Boutique, `checkoutservice.PlaceOrder` currently coordinates order placement synchronously over gRPC (`system/online-boutique/src/checkoutservice/main.go`). After successfully charging the customer's card (`paymentservice.Charge`, line 252) and shipping the items (`shippingservice.ShipOrder`, line 258), it calls `emailservice.SendOrderConfirmation` synchronously (lines 273–277). While an email failure is currently logged as a warning without failing the order, the synchronous gRPC call still blocks the checkout response.

Under the new requirement (R-12, `docs/scenario.md`), checkout must perform three post-order tasks:
1. Send an order confirmation email via **MailFast**, a third-party external provider replacing the mock `emailservice`.
2. Award loyalty points via a new `loyaltyservice`.
3. Record an "order placed" analytics event for the marketing team.
Marketing additionally expects further post-order actions next year (SMS notifications, fraud scoring, recommendations update).

The design must satisfy strict performance and operational constraints (`docs/scenario.md`):
- `checkoutservice.PlaceOrder` latency p95 must remain **≤ 500 ms** (currently ~300 ms).
- Peak traffic reaches **200 orders/s** during major promotional events (e.g. 11.11 sale).
- MailFast has a p95 latency of **1.5 s** (p99 of **8 s**), fails ~**0.5%** of requests, and enforces a strict rate limit of **50 emails/s**. Calling MailFast synchronously would immediately breach the 500 ms latency target and drop traffic during peak load (200 orders/s > 50 emails/s).
- Loyalty points must be awarded **within 1 minute** of the order, **exactly once** (no double crediting).
- An order must **never fail** because email, loyalty, or analytics is unavailable.
- The 4-person engineering team is proficient in Go and gRPC, and two members have production experience with **RabbitMQ**. All services run on Kubernetes, and company policy strongly prefers **open-source, self-hosted** infrastructure over cloud-managed services.

We must decide how to handle post-order operations without violating checkout latency, availability, and rate-limiting constraints.

## Decision drivers

1. **Performance (Latency):** `PlaceOrder` p95 response time must stay ≤ 500 ms under a peak load of 200 orders/s, despite MailFast's 1.5 s p95 latency.
2. **Availability & Fault Isolation:** Order checkout must never fail if post-order services (MailFast, `loyaltyservice`, analytics) are degraded or down.
3. **Throughput & Rate Limiting (Elasticity):** Peak traffic of 200 orders/s must be buffered and dispatched to MailFast at or below its 50 emails/s rate limit without message loss.
4. **Reliability & Consistency:** Loyalty points must be awarded within 1 minute, guaranteed at-least-once with idempotency to ensure exactly-once business outcome.
5. **Modifiability & Extensibility:** Post-order actions must be decoupled so that adding SMS, fraud scoring, and recommendations next year requires no code changes or redeployments to `checkoutservice`.
6. **Simplicity & Team Constraints:** Solution must run on Kubernetes as open-source software and fit the team's skillset (4 developers; 2 with RabbitMQ experience).

## Considered options

1. **Direct synchronous / parallel gRPC calls from `checkoutservice`**
2. **Asynchronous event-driven publish-subscribe with RabbitMQ**
3. **Distributed event streaming with Apache Kafka**

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Direct synchronous / parallel gRPC calls | • No new infrastructure to deploy or manage on Kubernetes.<br>• Reuses existing Go and gRPC skills across the team.<br>• Simple call chain and immediate debugging. | • Breaches latency goal: MailFast p95 of 1.5 s and p99 of 8 s directly blocks checkout beyond the 500 ms limit.<br>• Violates rate limit: Peak load of 200 orders/s overwhelms MailFast's 50 emails/s limit.<br>• Fragile availability: If fired in background goroutines, in-flight jobs are lost on pod restart/crash (violating loyalty delivery guarantees).<br>• Poor modifiability: Adding SMS, fraud scoring, and recommendations requires continuously modifying and redeploying `checkoutservice`. |
| RabbitMQ message broker (Publish-Subscribe) | • Keeps checkout fast: Publishing an `OrderPlaced` AMQP message takes < 5 ms, keeping `PlaceOrder` p95 ~305 ms (well below 500 ms limit).<br>• Buffers surges and enforces rate limits: Queues absorb 200 orders/s peak and throttle consumer delivery to MailFast at ≤ 50 emails/s.<br>• Fault isolation: Downstream downtime (MailFast, loyalty, analytics) does not impact checkout; durable queues retry deliveries.<br>• High modifiability: New downstream consumers (SMS, fraud, recommendations) attach new queues to the topic/fanout exchange without changing `checkoutservice`.<br>• Team fit: 2 of 4 developers already know RabbitMQ; runs natively on Kubernetes as an open-source component without cloud lock-in. | • Adds a stateful component (RabbitMQ cluster) to operate, monitor, and back up on Kubernetes.<br>• Introduces eventual consistency for post-order confirmation and loyalty points.<br>• Requires explicit consumer-side deduplication (idempotency key based on `order_id`) to prevent duplicate loyalty points on redeliveries. |
| Apache Kafka event streaming | • High throughput, durable append-only log with event replay capabilities.<br>• Independent consumer group offsets for loyalty, email, and analytics.<br>• Open-source and self-hostable on Kubernetes. | • Substantial operational overhead and resource footprint (KRaft/ZooKeeper, JVM tuning, partition rebalancing) for a 4-person team.<br>• Heavy overengineering for 200 orders/s peak volume.<br>• No prior team experience with Kafka operations.<br>• Complex per-consumer rate limiting and individual message acknowledgement compared to RabbitMQ's native AMQP queue prefetch and dead-letter exchanges. |

## Decision

Adopt **Option 2: Asynchronous event-driven publish-subscribe with RabbitMQ**.

Upon completing payment and shipping in `checkoutservice.PlaceOrder`, `checkoutservice` will publish an `OrderPlaced` event payload (containing `order_id`, customer email, order items, and total amount) to a durable RabbitMQ exchange (`order.events`). `checkoutservice` then immediately returns the `PlaceOrderResponse` to the client.

Individual durable queues will bind to the exchange for each business capability:
- `email-service-queue`: `emailservice` consumes messages with a QoS prefetch limit tuned to process at ≤ 50 emails/s to respect MailFast's rate limit, employing exponential backoff and a dead-letter exchange (DLX) for failed calls.
- `loyalty-service-queue`: `loyaltyservice` consumes events and awards points within seconds (< 1 minute requirement), recording processed `order_id` values to ensure exactly-once crediting.
- `analytics-service-queue`: A marketing worker ingests events into the marketing analytics datastore.
- Future services (SMS, fraud scoring, recommendations) will declare and bind new queues to the existing exchange with zero modifications to `checkoutservice`.

RabbitMQ will be deployed as an open-source high-availability cluster (Quorum Queues) on Kubernetes.

## Consequences

- **Positive:** `checkoutservice.PlaceOrder` response latency is decoupled from external and secondary systems, maintaining p95 latency well within the ≤ 500 ms target (**Performance**).
- **Positive:** Peak order bursts of 200 orders/s are safely buffered in RabbitMQ and throttled to stay within MailFast's 50 emails/s rate limit (**Throughput / Elasticity**).
- **Positive:** Outages or slowdowns in MailFast, `loyaltyservice`, or analytics do not impact checkout completion (**Availability / Fault Tolerance**).
- **Positive:** Adding new post-order handlers next year (SMS, fraud scoring, recommendations) requires only binding new queues to the exchange, keeping `checkoutservice` unmodified (**Modifiability**).
- **Positive:** Reuses existing team expertise (2 developers with RabbitMQ experience) and adheres to company infrastructure policy by using open-source, self-hosted Kubernetes software (**Cost / Portability**).
- **Negative:** Introduces a stateful infrastructure dependency requiring Kubernetes deployment, storage management, monitoring, and operational maintenance by the 4-developer team (**Operational Complexity**).
- **Negative:** Post-order processing becomes eventually consistent; confirmation emails and loyalty points appear with a brief asynchronous delay rather than synchronously on checkout completion (**Consistency**).
- **Negative:** Network redeliveries and at-least-once broker delivery require downstream services (specifically `loyaltyservice`) to implement idempotency handling (**Modifiability / Complexity**).
- **Risks and mitigations:**
  - *Risk:* RabbitMQ cluster outage could prevent `checkoutservice` from publishing events, delaying post-order processing.
    *Mitigation:* Deploy RabbitMQ with Quorum Queues across 3 Kubernetes nodes with persistent volume storage. In `checkoutservice`, if publishing fails after short retries, log the failure with high severity and implement a local fallback buffer or graceful degradation so customer order placement remains successful.
  - *Risk:* Large backlog in `email-service-queue` during 11.11 peak (200 orders/s arriving vs 50 emails/s processed creates an accumulating queue of 150 msg/s).
    *Mitigation:* A 10-minute 200 orders/s surge generates 120,000 messages, requiring 40 minutes to drain at 50 emails/s. RabbitMQ easily buffers hundreds of thousands of messages on disk without memory exhaustion. Configure alert thresholds for consumer lag and inform marketing/customer support of peak email delivery delays.
  - *Risk:* Duplicate message processing causing customers to receive double loyalty points.
    *Mitigation:* Require `loyaltyservice` to enforce deduplication by storing `order_id` in a database table with a unique constraint as part of the loyalty credit transaction.

## Validation

- **Load testing:** Execute Locust load tests generating 200 orders/s peak load against `checkoutservice.PlaceOrder`. Verify that `PlaceOrder` p95 latency remains ≤ 500 ms and that all messages are published to RabbitMQ.
- **Rate-limit verification:** Monitor outbound HTTP calls from `emailservice` to MailFast during peak load tests to verify dispatch does not exceed 50 emails/s.
- **SLA & Consistency verification:** Measure the time from order completion to loyalty point allocation during peak load, verifying that 100% of loyalty points are awarded within 60 seconds.
- **Fault-injection testing:** Simulate MailFast API errors (0.5% failure rate and simulated outages) and `loyaltyservice` pod crashes; confirm that zero orders fail at checkout and all post-order events are successfully processed upon service recovery.
- **Idempotency testing:** Replay identical `OrderPlaced` messages to `loyaltyservice` and verify no duplicate points are awarded.

## Open questions

- What data store will `loyaltyservice` use to persist loyalty balances and maintain the unique `order_id` record for idempotency?
- What are the SLA and invocation timing requirements for the upcoming fraud scoring service? (e.g. Does fraud scoring need to run synchronously prior to payment capture, or can it run asynchronously post-order as an audit/flagging mechanism?)
- What is the retention and alert escalation policy for the dead-letter queue (DLQ) when MailFast permanently rejects an email (e.g. invalid recipient address)?
- What schema format (e.g. Protocol Buffers, CloudEvents JSON) is preferred for the `OrderPlaced` message payload across current and future post-order consumers?
- In the event of a total RabbitMQ broker outage, should `checkoutservice` fail the order or proceed with order placement and log an alert for manual recovery?
