[← Ondřej Maléř · all projects](https://ondmal.github.io)

# Multi-Party Document Signing Service

A backend service that turns an **external signing platform** into a
**multi-signatory signing capability** for any consuming service. It hides the
platform behind one stable contract and adds what the platform lacks: a signing
process across several signatories that succeeds only when **every one of them
signs**. In production, used by two consuming services.

**Role:** solution architect and senior developer — designed the whole system
and implemented the most critical parts of it.

**System of record for the *multi-signatory process* — not for the single
session, and not for the signed documents.** The signing platform owns each
**signing session** — one signatory, their identity verification, their
signature. The consumer owns the **documents**. The service owns only the
**process** above those sessions — which signatories, which documents, the
all-or-nothing outcome — and holds the signed documents just until the
consumer collects them.

---

## The problem

Several business flows need documents — contracts, typically — signed digitally
by **more than one party** with an **Advanced Electronic Signature (AES)** under
eIDAS: the signatory's identity verified against an ID document, the signature
uniquely linked to that person, and the result legally valid. An **external
signing platform** was the
given precondition: it provides the signatory's user interface, the identity
verification and the cryptographic signing, including the organisation's
counter-signature. But it works with **one signatory per session**, and it knows
nothing about the business flow that needs the signature.

So what had to be built is a service that:

1. **Composes single-signatory sessions into one multi-signatory process** with
   all-or-nothing semantics — restarts, refusals, expiry, cancellation and a
   time limit included.
2. **Serves any consuming service through one contract**, while leaving each
   consumer's business decisions — *is this the right person?*, *which documents
   exactly?* — with that consumer.

The hard part lies between the systems rather than in either of them. Progress
is reported by the platform asynchronously, and its notifications are not
guaranteed to arrive; at two points a signatory sits in the signing session
waiting while the service fetches an answer from the consumer; and a signature
is a legal act — a process must never be lost, stuck, or completed twice, and
personal data must not outlive its purpose.

---

## System context

The service sits between the consuming services and the signing platform: the
consumers create and follow signing processes through its API and answer its
callbacks; the platform runs each signatory's session and reports progress by
webhook; invitations go out by e-mail through a notification service.

![System context: consuming services call the signing service and answer its callbacks; the service drives the external signing platform, which the signatories use and which reports back by webhook; around them sit the identity provider, the notification service and the observability backend](images/multi-party-document-signing-system-context.svg)

---

## Technology

| Area | Choices |
|---|---|
| Language / platform | C# / .NET |
| Style | Clean Architecture + DDD; asynchronous, command-driven processing |
| Persistence | PostgreSQL via EF Core; Azure Blob Storage for signed documents |
| Hosting / runtime | Azure; Kubernetes (all workloads containerised via one Helm chart) |
| Async & messaging | Azure Functions worker (queue- & timer-triggered) on Azure Storage Queues; transactional outbox |
| Integration | REST adapters with centralised resiliency (timeouts, retries, circuit breakers); webhooks in, callbacks out |
| Security | OAuth / OpenID Connect with role-based access, enforced at the API gateway; workload identity to Azure resources; webhooks protected by IP allow-listing at the external gateway, HMAC signatures and Basic auth |
| Observability | OpenTelemetry (traces, metrics, logs) exported to Honeycomb; dashboards & SLOs as code |
| Testing | Unit and integration tests (containerised PostgreSQL and storage, mocked external systems) |

---

## Highlights

- **Multi-signatory signing on a single-signatory platform.** One process,
  several independent sessions, all-or-nothing outcome — behind a facade that
  keeps the platform replaceable. → *The signing process*, *Design decisions*
- **One service for any consumer.** A single contract with consumer-owned
  decisions. → *Design decisions*
- **Reliable asynchronous processing.** Transactional outbox, idempotent
  commands, replay with backoff and recovery of missed webhooks — no process is
  lost or left stuck. → *Design decisions*, *The signing process*
- **A process worker, not a document store.** No identity-document data kept;
  signed documents and personal data removed after 30 days, the rest after a
  year. → *Design decisions*
- **Observable, tested, documented.** OpenTelemetry end to end, SLOs on the
  platform integration, unit and integration tests, and a full documentation
  set. → *Observability*, *Quality & delivery*

---

## Design decisions

### Shape

**Reliability over scale.** Traffic is low — tens to a hundred signing processes
a day, tens of thousands of API requests, a few thousand processed commands —
and the domain does not need more. Correctness comes first — nothing lost,
stuck, or completed twice — and every choice below favours it over raw
throughput. Parallelism follows the domain: a signing
session is sequential by nature — each step follows the signatory's own action —
so commands for one process run one at a time, while different processes run in
parallel. That makes the process the natural partition key: the API hosts scale
out today, and the worker can scale out by partitioning the queue per process.
Scale was never the driver, so that partitioning is designed in but not built —
it costs nothing until the numbers ask for it.

![Scaling model: within one signing process each session is a sequential chain of steps that follows the signatory, and the process runs one command at a time; many processes are independent and run in parallel; so the process id partitions the command queue, with one worker per partition](images/multi-party-document-signing-scaling-model.svg)

**DDD and Clean Architecture rather than vertical slices.** Signing is not a set
of independent features: every use case — create, start, confirm, sign, restart,
cancel, expire — operates on the same process aggregate under the same rules:
session states, restart limits, the all-or-nothing outcome. Vertical slices as
the architecture would scatter those rules across slices and couple each one to
infrastructure. Keeping a single infrastructure-free domain, wrapped by
Application and Infrastructure, puts the whole signing logic in one testable
place and lets two hosts run the same core; use cases are still grouped by
feature on top of it.

**An existing signing platform, behind a facade.** The external signing platform
was already in place, under contract, so building on it was a given rather than
choosing a mainstream e-signature service. The facade keeps that reversible:
consumers see only the service's own contract, never the platform's, so another
provider can be implemented as a new adapter without affecting them.

**One service for any consumer.** The service knows signatories, documents and
sessions — never *why* something is being signed. Each consumer keeps its own
business decisions: it confirms whether a verified identity matches its
customer, and it prepares the documents only once the signatory is identified,
so they can carry data taken from the identity document. Consumers are a
registered set with per-consumer configuration — callback endpoints and the
status changes each one wants — so adding one is a small, contained change. The
cost: every consumer must implement the callback contract.

**Not the document's system of record — so nothing is kept.** The service runs
the signing *process*; the consumer that asked for the signature owns the result
and its long-term storage, so the service holds signed documents only until the
consumer collects them: after 30 days they are deleted and signatory names and
e-mail addresses are overwritten;
what remains — identifiers, statuses, timestamps, counts — serves reporting and
is deleted after a year, all automatically on a timer. Identity-document data is
never stored at all: it exists only inside the encrypted audit record the
platform embeds in the signed PDF, readable by the organisation alone, and
unsigned documents only pass through. Keeping less is also the simplest way to
stay compliant with data-protection rules. The cost: a consumer must collect its
documents within the retention window.

### Integration

**Callbacks to the consuming services, not events.** At the two points where
the signatory is waiting inside the signing session (see *The signing process*),
the service gets the consumer's answer with a direct request–response call; an
asynchronous exchange would work too, but would turn a question with a human
waiting for the answer into two messages, a correlation and a timeout — more to
fail, for no gain. A slow consumer is retried within a short budget and the step
replayed; but a signatory will not wait more than a minute or two, so
availability at these two points is the consumer's responsibility — the session
then expires on the platform and can be restarted. Every status change reaches
the consumer through the same callback channel, so the consumer sees the whole
signing lifecycle and can publish it onwards. What is missing is a stream
published by the service itself, for auditing, reporting or other observers
that are not its consumers; no requirement asked for one. Closing the gap is
cheap — the same outbox that carries the callbacks can carry events — and
events would complement the callbacks, not replace them.

**Webhooks, with a recovery check behind them.** The platform reports progress
by webhook, and webhook delivery is not guaranteed. Polling alone would add
latency and load on a platform that allows only a few concurrent calls;
webhooks alone would let one lost call strand a signatory. So the service
relies on webhooks
and, after each step that expects one, schedules a deferred **recovery check**:
if the webhook has not arrived, the check asks the platform for the session's
state and creates the missing steps itself. A late webhook arriving after a
recovery is recognised as a duplicate and ignored.

**A separate internet-facing webhook receiver.** Only a small receiver host is
exposed to the internet, behind the external gateway, which admits only the
platform's IP addresses. The receiver verifies the caller — an HMAC signature,
Basic auth — translates the webhook, forwards it to the service over the
internal gateway under its own identity, and answers at once. The service itself stays internal. The cost: one more
deployable and one more hop.

### Mechanics

**Transactional outbox, replay and idempotency.** Every command is written to an
outbox table in the same transaction as the state change it follows from,
published to the queue right after commit, and recovered by a timer-driven
publisher if that publish failed. On the worker, commands for one signing
process run one at a time, any write that still races another is caught by an
optimistic concurrency check, and duplicates are filtered by command id. A
command that fails — a transient error, a write conflict, or a step that
arrived before its prerequisite — is replayed through the outbox with
exponential backoff. No state change commits without its follow-up work, and no
follow-up runs without its state change.

![Life of one command: commit the state change and its commands in one transaction, publish to the queue after commit, execute serialised per process on the worker, call the signing platform or a consumer, and turn the platform's webhook into the next command; safety nets are the outbox publisher, replay with backoff, the recovery check and the lifecycle timer, with dead-lettering for expired messages](images/multi-party-document-signing-async-command.svg)

**Storage Queues rather than a message broker.** The transport needs no
ordering — the worker serialises per process and an early step is detected and
replayed — and no scheduling, which the outbox provides. A plain queue is
sufficient; a broker would add cost without a capability the service uses.

**No Redis.** The duplicate filter is a per-instance in-memory cache, and
correctness never depends on it: handlers are idempotent and transitions are
guarded. Redis would add an operational dependency without adding a guarantee;
a Redis-backed filter exists for the day it is worth it.

---

## Component view

![Component view: the external signing platform calls the webhook receiver, consuming services call the WebApi host; the receiver forwards to the WebApi host, and the WebApi and command-processor hosts share one Clean Architecture core of Infrastructure, Application and Domain, depending on PostgreSQL, Blob Storage, Storage Queues and the external systems](images/multi-party-document-signing-component-view.svg)

**One core under the hosts.** **Domain** (the signing process aggregate, its
sessions and rules, free of infrastructure) at the centre, wrapped by
**Application** (use cases and command handlers), then **Infrastructure**
(persistence, queues, storage, external adapters); dependencies point inward.

**Hosts.** The **WebApi host** serves the consumer API — published through an
internal API gateway that enforces OAuth and roles — the internal webhook
endpoint and read-only support endpoints. The **command-processor host** runs
every command through the same Domain → Application → Infrastructure stack and
owns the timers. The **webhook receiver** is deliberately thin: it depends only
on the service's API client, never on its core.

**Adapters and the consumer contract.** Each external system sits behind a port,
and each adapter is wrapped in the same layers — tracing, error mapping — over
centralised timeouts, retries and circuit breakers. The callback contract every
consumer implements is published as a contracts package with a client, so the
service calls all consumers the same way.

---

## The signing process

![Signing process: the consumer creates a process; the service starts one session per signatory on the signing platform; the signatory verifies their identity, the consumer confirms it and prepares the documents while the signatory waits; the signatory signs, the platform counter-signs, the service keeps the signed documents; the process completes only when every signatory has signed, and the consumer is notified and collects the documents](images/multi-party-document-signing-signing-process.svg)

- **One call starts it, everything else follows.** The consumer creates a
  process — signatories and document types — and gets its id back at once. From
  there the service drives one **session per signatory** on the platform; each
  signatory gets an invitation by e-mail, and sessions run independently, in
  any order.

- **Two answers only the consumer can give.** When the platform has verified a
  signatory's ID, the service asks the consumer whether the identity matches its
  customer, and passes the verdict back to the platform. Once confirmed, it asks
  the consumer for the documents to sign, verifies their integrity hashes, and
  hands them to the platform — both while the signatory waits (see *Design
  decisions*). The platform then has the signatory sign, adds the
  organisation's counter-signature, and the service downloads the signed
  documents and keeps them.

- **All or nothing.** The process completes only when every session has signed.
  A failed session can be restarted by the consumer, a limited number of times;
  any refusal, or a session that has run out of restarts, fails the whole
  process. The consumer can cancel at any time, and an unfinished process is
  cancelled automatically after five days. Every change of a session or of the
  process is reported to the consumer by callback; the consumer then collects
  the signed documents.

- **Nothing lost, nothing stuck.** All background work runs as commands through
  the transactional outbox (see *Design decisions*). Delivery is at-least-once
  and commands are idempotent, so a replayed command or a duplicated webhook
  converges to the same result — a step that no longer matches the session's
  state is ignored, so a process cannot complete twice or notify its consumer
  twice. A missed webhook is recovered by the deferred recovery check. Messages
  that outlive their time-to-live are dead-lettered as explicit data-loss
  events, never dropped silently.

---

## Runtime & infrastructure

Deployed on **Azure**, running on **Kubernetes** — all workloads are
containerised and rolled out together via one Helm chart.

- **Three workloads, one database.** The WebApi host and the webhook receiver
  run as plain containers with autoscaling; the command processor is an Azure
  Functions container with the queue-triggered command handler and timers for
  the outbox publisher, outbox clean-up and process lifecycle. All share a
  managed PostgreSQL.
- **Storage and identity.** An Azure Storage Account provides the command queue,
  the signed-document blobs and the leases the Functions host needs. Access to
  the database, storage and Key Vault uses workload identity — no stored
  passwords for Azure resources.
- **Safe rollouts.** A per-command time cap is sized below the queue's
  redelivery window and the pod's shutdown grace period, so during a rollout an
  in-flight command either finishes or is cleanly scheduled for replay.

---

## Observability & operability

- **OpenTelemetry end to end** — traces, metrics and logs with a consistent
  span-and-baggage convention and propagated context, so one signing process
  reads as a single connected story across the webhook receiver, the API, the
  worker and the calls to the platform and the consumers.
- **Business and technical metrics** — process and session lifecycle, command
  outcomes, outbox backlog and age — with dashboards, **SLOs** on the platform
  integration (reliability and webhook liveness) and alerts, all kept as code.
- **Built for support** — a command history records every processed command
  with the entity state before and after, exposed through read-only support
  endpoints; a health check flags processes the lifecycle timer should already
  have ended.

---

## Quality & delivery

- **Test strategy**: unit tests on the infrastructure-free domain and on the
  application, and integration tests against containerised PostgreSQL and
  storage with mocked external systems — around a thousand automated tests in
  all.
- **A scenario-driven stand-in for the signing platform**, selectable per
  request outside production, so consumers can exercise signed, failed and
  stuck outcomes against the real service.
- **Fully documented**: an architecture overview with recorded decisions, a
  contract specification and a **consumer integration guide**, and operational
  docs (deployment, observability, test strategy) — so a new consuming service
  can integrate, and the system can be operated, from the docs alone.
