Agent: Antigravity | Model: Gemini 3.8 Flash

# ADR-0002: Use Asynchronous Event-Driven Architecture for Post-Order Processing

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Architecture & Engineering Teams [Inferred]

## Context and problem

After an order is placed, Online Boutique must execute three post-order operations:
1. Send an order confirmation email via **MailFast**, a new external email provider replacing the mock `emailservice` logic.
2. Award loyalty points via a new `loyaltyservice` (to be built).
3. Record an "order placed" analytics event for the marketing team.

Furthermore, marketing expects to add more post-order actions next year, including SMS notifications, fraud scoring, and a recommendations update.

Specific quantitative constraints, performance targets, SLA requirements, and existing infrastructure details are not evidenced in the provided files. The architectural challenge is how to coordinate these heterogeneous, expanding post-order actions without degrading order placement performance or coupling checkout to third-party availability.

## Decision drivers

1. **Modifiability / Extensibility:** The system must easily accommodate future post-order actions (SMS, fraud scoring, recommendation updates) without altering core checkout logic.
2. **Availability / Resilience:** Failure or latency in third-party services (MailFast) or secondary tasks (analytics, loyalty) must not block or fail the primary order placement flow.
3. **Performance (Latency):** Order placement response time experienced by the customer must remain minimal and not be compounded by post-order execution times.
4. **Operability & Complexity:** Implementation and maintenance overhead should remain manageable for the engineering team.

## Considered options

1. **Option 1: Synchronous Direct RPC/HTTP Orchestration** — The checkout service calls MailFast (or an updated `emailservice`), `loyaltyservice`, and an analytics endpoint sequentially or concurrently via synchronous RPC/HTTP requests before returning success to the user.
2. **Option 2: Asynchronous Publish-Subscribe Event Broker** — Upon order placement, the checkout service publishes an `OrderPlaced` domain event to a central publish-subscribe message broker. Downstream consumers (`emailservice`/MailFast adapter, `loyaltyservice`, analytics worker, and future subscribers) independently subscribe and process events asynchronously.
3. **Option 3: Asynchronous Centralized Task Queue / Job Worker** — The checkout service enqueues a post-order background job into a task queue. A centralized worker service dequeues the job and sequentially/concurrently invokes the individual post-order integrations.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| **Option 1: Synchronous Direct RPC/HTTP Orchestration** | - Simple conceptual model and minimal operational infrastructure (no message broker needed).<br>- Straightforward local tracing and immediate failure detection. | - Poor modifiability: Adding future actions (SMS, fraud, recommendations) requires modifying and redeploying the checkout service.<br>- Poor availability: Checkout availability depends directly on external MailFast API and secondary service uptimes.<br>- Poor performance: Order completion latency increases with every downstream network call. |
| **Option 2: Asynchronous Publish-Subscribe Event Broker** | - High modifiability: New post-order consumers can subscribe to `OrderPlaced` events independently with zero changes to checkout service.<br>- High availability: Failure or downtime in MailFast or analytics does not impact order placement; messages persist in broker for retry.<br>- High performance: Checkout responds immediately after publishing an event, decoupling checkout latency from post-order execution. | - Higher operability and infrastructure overhead: Requires deploying, monitoring, and operating a message broker.<br>- Eventual consistency: Actions occur asynchronously, requiring idempotent event handling in consumers to handle retries and duplicates. |
| **Option 3: Asynchronous Centralized Task Queue / Job Worker** | - Good performance: Checkout offloads work asynchronously to a queue, keeping user-facing latency low.<br>- Centralized retry and rate-limiting policies for tasks. | - Moderate modifiability: The centralized worker or task definition must still be updated whenever new post-order actions are introduced.<br>- Single point of failure/bottleneck in the centralized task worker service.<br>- Operability overhead of managing task queue infrastructure. |

## Decision

Adopt **Option 2: Asynchronous Publish-Subscribe Event Broker**.

Publishing an `OrderPlaced` domain event allows Online Boutique to fully decouple the core checkout transaction from secondary and third-party integrations. This satisfies the primary driver (**Modifiability**), enabling seamless addition of upcoming post-order features (SMS, fraud scoring, recommendation updates) as independent subscribers without modifying or redeploying the checkout service. It also satisfies **Availability** and **Performance** by isolating the user-facing checkout response from external MailFast latency and third-party downtime.

## Consequences

- **Positive:**
  - High modifiability: New post-order actions (SMS, fraud scoring, recommendations) can be added as isolated subscribers without changes to the checkout service (quality attribute: modifiability).
  - High availability: Outages or slowdowns in MailFast, `loyaltyservice`, or analytics do not fail the customer's checkout transaction (quality attribute: availability).
  - Low user-facing latency: Checkout response completes immediately upon publishing the event without waiting for external API calls (quality attribute: performance).
  - Fault tolerance: Failed post-order actions can be retried independently using broker retry queues and dead-letter queues (quality attribute: reliability).

- **Negative:**
  - Increased operational overhead: Engineering must provision, monitor, and maintain message broker infrastructure (quality attribute: operability, cost).
  - Eventual consistency: Confirmation emails, loyalty points, and analytics are processed asynchronously rather than instantaneously at order submission (quality attribute: consistency).

- **Risks and mitigations:**
  - *Risk:* Duplicate message delivery causing duplicate loyalty points or multiple confirmation emails.
    *Mitigation:* Implement idempotent message handling in all consumer services using the unique order identifier (`order_id`) (quality attribute: reliability).
  - *Risk:* Message broker downtime causing order event drops.
    *Mitigation:* Deploy the broker in a highly available cluster and consider an outbox pattern in the checkout data layer if persistent delivery guarantees are strictly required (quality attribute: availability).
  - *Risk:* MailFast API rate limiting or temporary outages causing email delivery backlogs.
    *Mitigation:* Implement rate-limited consumers with exponential backoff retries and dead-letter queues (DLQ) for failed dispatches (quality attribute: reliability).

## Validation

- **End-to-end integration testing:** Verify that placing a test order publishes an `OrderPlaced` event and successfully triggers MailFast email dispatch, loyalty point crediting, and analytics ingestion.
- **Latency benchmark:** Run load tests on checkout to verify that p95/p99 checkout response times remain unchanged regardless of downstream MailFast response latency.
- **Fault-injection testing:** Simulate MailFast API downtime and verify that orders succeed and email events are queued and reprocessed once MailFast is restored.

## Open questions

1. **Volume and performance metrics:** What are the expected order volumes (peak orders/second) and SLA requirements for post-order actions (e.g., maximum acceptable delay for order confirmation emails)? (Not evidenced in the provided files)
2. **Existing infrastructure & broker choice:** Does Online Boutique already have a messaging system in place (e.g., RabbitMQ, Kafka, NATS, or cloud-native pub/sub), or does new infrastructure need to be selected and provisioned? (Not evidenced in the provided files)
3. **Deployment environment & hosting:** What platform is Online Boutique deployed on (e.g., Kubernetes, cloud provider), and are managed message brokers available? (Not evidenced in the provided files)
4. **Team operational expertise & cost constraints:** What are the team's operational skills and budget for running and supporting message broker infrastructure? (Not evidenced in the provided files)
5. **MailFast API specifications:** What are the authentication methods, rate limits, latency profiles, and webhook/retry mechanisms of MailFast? (Not evidenced in the provided files)
6. **Interface specifications for new components:** What are the API/schema contracts and storage backends for `loyaltyservice` and the marketing analytics destination? (Not evidenced in the provided files)
7. **Transactional delivery requirements:** Is at-least-once delivery with consumer deduplication sufficient, or is a transactional outbox pattern required to guard against crash-between-order-save-and-event-publish? (Not evidenced in the provided files)
