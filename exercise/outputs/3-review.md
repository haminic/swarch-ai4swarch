Agent: Antigravity | Model: Gemini 3.8 Flash (High)

# Architecture Review: ADR-0002 (RabbitMQ for Asynchronous Post-Order Event Processing)

## 1. Quality-attribute scenarios

- **Scenario 1: Checkout Latency & Peak Load (Performance)**
  - *Source:* Customer web browser / Frontend client.
  - *Stimulus:* 200 orders/s burst during promotional events (e.g., 11.11 sale).
  - *Environment:* MailFast external API experiences high latency (p95 = 1.5 s, p99 = 8 s).
  - *Response:* `checkoutservice.PlaceOrder` executes core payment and shipping, publishes an `OrderPlaced` event asynchronously to RabbitMQ, and returns immediately without awaiting downstream tasks.
  - *Response Measure:* `checkoutservice.PlaceOrder` p95 response time remains ≤ 500 ms (measured at ~305 ms).

- **Scenario 2: Provider Rate Limiting & Buffer Elasticity (Elasticity / Reliability)**
  - *Source:* High-volume post-order event publishing.
  - *Stimulus:* Sustained peak traffic of 200 orders/s over a 10-minute burst (120,000 total orders).
  - *Environment:* MailFast enforces a strict rate limit of 50 emails/s and fails ~0.5% of requests.
  - *Response:* RabbitMQ absorbs the surge in durable queues; `emailservice` consumers pull messages using tuned QoS prefetch to throttle delivery; failed calls are retried via backoff and dead-letter exchange (DLX).
  - *Response Measure:* Outbound requests to MailFast never exceed 50 emails/s; zero messages are dropped; queue drains fully within ~40 minutes.

- **Scenario 3: Fault Isolation & Exactly-Once Crediting (Availability / Consistency)**
  - *Source:* Customer checkout execution.
  - *Stimulus:* Order placed while external MailFast is down or network disconnects during event consumption.
  - *Environment:* Downstream post-order services experience transient outages; RabbitMQ redelivers unacknowledged messages.
  - *Response:* Core order succeeds without error; `loyaltyservice` processes redelivered events within an ACID transaction against PostgreSQL using `order_id` deduplication.
  - *Response Measure:* 100% of customer orders succeed at checkout; loyalty points are credited within 1 minute of order placement; exactly-once points awarded (0 duplicate credits).

---

## 2. Findings table

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | *Dual-write failure without transactional outbox:* `checkoutservice` charges payment and ships before publishing to RabbitMQ in-memory. If the broker is unreachable or the pod restarts mid-flight, the order completes but post-order events are lost. | Billed customers will not receive confirmation emails or loyalty points, violating R-12 reliability guarantees and requiring manual reconciliation. |
| **Risk** | *Misclassification of fraud scoring as post-order async event:* The ADR plans to add fraud scoring next year simply as another downstream consumer queue on the post-order exchange. | Fraud scoring usually needs to evaluate risk *before* payment capture or shipment. Running it asynchronously post-shipping allows fraudulent orders to be fulfilled before fraud is flagged. |
| **Risk** | *Operational overhead for a 4-person team:* Operating self-hosted, clustered RabbitMQ (Quorum Queues) and PostgreSQL on Kubernetes introduces complex stateful management (PVCs, backup/restore, operator upgrades, split-brain mitigation). | Strains a small team of 4 developers, risking production outages or configuration drift if Kubernetes cluster maintenance lags. |
| **Non-risk** | *Decoupling MailFast via message buffering:* Offloading email dispatch to an asynchronous RabbitMQ queue isolates checkout from MailFast's latency (1.5 s p95, 8 s p99) and 0.5% error rate. | Guarantees `checkoutservice.PlaceOrder` satisfies the ≤ 500 ms SLA and protects against hitting MailFast's 50 emails/s rate limit during 200 orders/s surges. |
| **Non-risk** | *Idempotency via PostgreSQL unique constraint:* `loyaltyservice` wraps point crediting and `order_id` insertion in an ACID transaction with a `UNIQUE` constraint. | Eliminates double-crediting risks caused by AMQP at-least-once message redeliveries or network reconnects, satisfying the exactly-once business requirement. |
| **Sensitivity point** | *Consumer concurrency and prefetch tuning:* Outbound rate to MailFast depends directly on the number of `emailservice` replicas multiplied by worker concurrency and prefetch count. | If horizontal pod autoscaling (HPA) or worker concurrency scales unchecked, outbound requests will breach the 50 emails/s cap, triggering HTTP 429 rate-limit rejections. |
| **Sensitivity point** | *Burst duration vs. email delivery delay:* Arrival at 200 msg/s against 50 msg/s drain rate accumulates an unread backlog at 150 msg/s. | A 10-minute surge produces a 40-minute queue drain time; customers ordering at the end of the peak wait up to 40 minutes for confirmation emails, directly impacting customer support. |
| **Trade-off point** | *Latency vs. Operational complexity:* Asynchronous messaging achieves ≤ 500 ms p95 response time and decoupled extensibility, but introduces a distributed stateful broker. | Delivers required performance and modifiability at the expense of adding Kubernetes infrastructure maintenance to a 4-developer team. |
| **Trade-off point** | *Fault isolation vs. Immediate consistency:* Returning success immediately decouples checkout from downstream failures, but makes email confirmation and loyalty crediting eventually consistent. | Protects checkout conversion rates, but users cannot immediately see updated loyalty balances or receive instant confirmation upon checkout completion. |

---

## 3. Missing or weak

- **Options not considered:**
  - *Transactional Outbox / CDC or Redis Streams:* Online Boutique already runs Redis for cart storage (ADR-0001). Using Redis Streams or a transactional outbox table with Change Data Capture was not evaluated, jumping directly from synchronous gRPC to clustered RabbitMQ and Kafka.
- **Consequences not stated:**
  - *PostgreSQL write capacity under peak load:* While email dispatch is throttled to 50/s, `loyaltyservice` must process events within 1 minute, requiring up to 200 writes/s during peak load. The ADR does not analyze PostgreSQL connection pooling, storage IOPS, or Kubernetes resource sizing.
  - *Dead-letter queue (DLQ) operations and poison pills:* The ADR mentions a DLX but specifies no retention policy, alerting thresholds, or tooling to inspect, fix, and replay poison-pill messages (e.g., malformed email addresses).
- **Unsupported or likely wrong claims:**
  - *Extensibility for Fraud Scoring:* Claiming future fraud scoring can be added without changing `checkoutservice` is architecturally unsound if fraud checks must block fraudulent transactions prior to authorization.
  - *Ephemeral fallback buffer claim:* The mitigation claims `checkoutservice` can "implement a local fallback buffer" if RabbitMQ fails; an in-memory buffer inside an ephemeral Kubernetes pod cannot survive pod evictions or restarts without data loss.

---

## 4. Verdict

**Verdict:** **Accept with changes**

**Three most important changes:**
1. **Specify a reliable publishing mechanism in `checkoutservice`:** Define how `checkoutservice` guarantees event delivery even if RabbitMQ is temporarily down or a pod crashes after payment capture (e.g., publisher confirms with a persistent outbox or durable local disk spool).
2. **Re-classify fraud scoring as a pre-order workflow:** Decouple fraud scoring from the post-order asynchronous event bus, planning it instead as a synchronous pre-payment/pre-shipping interceptor before order capture.
3. **Define a concrete operational baseline for RabbitMQ and PostgreSQL:** Provide a clear operations plan suited for a 4-developer team (e.g., RabbitMQ Cluster Kubernetes Operator, Quorum Queues with replication factor 3, automated volume snapshots, and Prometheus alerting on consumer lag and DLQ depth).
