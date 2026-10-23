I agree with your approach. Before writing any implementation, we should perform a build-versus-buy assessment for every module and identify the existing libraries, frameworks, and infrastructure that can fulfil its requirements.

The objective should be to write as little infrastructure code as possible while retaining control over the architecture, developer experience, and enterprise policies.

For your Java 21+ and Gradle platform, I would establish this rule:

- Adopt existing components for capabilities they already solve reliably.
- Integrate them behind a small platform API when teams need a consistent developer experience.
- Build custom code only for gaps in the existing ecosystem or for enterprise-specific conventions.
- Avoid implementing distributed systems primitives yourself unless there is a compelling, demonstrated requirement.

Your platform should be the layer that makes these components work together consistently, not a competing implementation of each one.

# 1. Module-by-module build-versus-buy assessment

This is the initial technology decision register I would use for your proposed platform. It is based on your original module list, with recommendations informed by the existing tools and official documentation reviewed above.

Status meanings: Adopt = use existing tool; Integrate = add a thin platform layer; Build = implement only a genuine gap; Defer = wait for a validated use case.

| Your module                  | Existing tool(s) to evaluate                                      | What your platform should own                                   | Decision          |
| ---------------------------- | ----------------------------------------------------------------- | --------------------------------------------------------------- | ----------------- |
| `bom`                        | Micronaut Platform BOM, Gradle `java-platform`, version catalogs  | Supported dependency versions and compatibility policy          | Integrate         |
| `domain-core`                | Java 21+, Jakarta Validation where needed                         | Only minimal shared primitives and conventions                  | Build selectively |
| `outbox`                     | Debezium Outbox Event Router; JDBC-based outbox alternatives      | A common API/configuration and selected delivery strategy       | Integrate         |
| `eventing`                   | Micronaut Kafka or another existing broker integration            | Common publishing conventions, headers, error policy            | Integrate         |
| `cqrs`                       | Plain Java, Micronaut DI; optional architecture/testing libraries | Minimal command/query conventions and handler wiring            | Build thin        |
| `observability`              | Micronaut Tracing, OpenTelemetry, Micrometer, Prometheus          | Default configuration, service metadata and standards           | Integrate         |
| `event-schema`               | CloudEvents Java SDK; schema registry if needed                   | Organization-specific event compatibility rules                 | Integrate         |
| `security`                   | Micronaut Security, existing identity provider                    | Permission conventions, tenant context and audit identity       | Integrate         |
| `search-projection`          | Elasticsearch/OpenSearch Java clients, broker integrations        | Projection lifecycle and common error-handling conventions      | Integrate         |
| `workflow`                   | Temporal Java SDK                                                 | A simpler supported API and organizational defaults             | Integrate         |
| `testing`                    | JUnit 5, Testcontainers, framework test utilities                 | Shared fixtures, test conventions and platform-specific helpers | Integrate         |
| `autoconfiguration`          | Micronaut configuration and dependency injection                  | Conditional wiring of your own platform modules                 | Build thin        |
| `micronaut-platform-starter` | Micronaut starter conventions                                     | A curated, optional bundle of platform capabilities             | Build thin        |
| `micronaut-ddd-starter`      | Micronaut plus architecture tests                                 | Project conventions and optional DDD scaffolding                | Build thin        |
| `micronaut-cqrs-starter`     | Your CQRS module                                                  | Wire up CQRS without imposing event sourcing                    | Build thin        |
| `micronaut-outbox-starter`   | Your chosen outbox integration                                    | Configure the selected outbox implementation                    | Build thin        |

For dependency management, Gradle explicitly distinguishes version catalogs from platforms: catalogs centralize dependency declarations, while platforms participate in dependency version alignment. Use both for their respective purposes.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.gradle.org\&sz=32)

Gradle Documentation

+1



The key point is that most of the value is in configuration, integration, and consistent contracts, not in implementing these technologies from scratch.

## 2. Exact recommendations for each module

### 2.1 `bom` — adopt Gradle's dependency management capabilities

No custom dependency engine

Use:

- Gradle `java-platform` for your published enterprise BOM.
- Gradle version catalog (`libs.versions.toml`) for aliases and convenient dependency declarations.
- Micronaut's BOM for framework-managed dependencies.
- Dependency locking and automated dependency-update checks for reproducible builds.

Your BOM should align the versions of your own modules and any dependencies that need additional organization-wide control. Avoid copying every Micronaut dependency version into your own BOM unnecessarily.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.gradle.org\&sz=32)

Gradle Documentation

+2



Build: your compatibility policy, tested version matrix and release process.

Do not build: a dependency resolution system.

### 2.2 `domain-core` — keep this deliberately small

Use Java 21+ and ordinary Java types. Records are useful for immutable value types where appropriate. Use Jakarta Validation only where validation annotations genuinely belong in the model or application boundary.

Potential contents:

- A small domain-event contract, if needed.
- Explicit domain identifiers or other primitives only if they provide real shared value.
- Minimal domain error conventions, if consistency is required.

Do not include Micronaut annotations, database entities, persistence abstractions, generic repositories, or aggregate base classes merely to standardize the architecture.

Build selectively: shared primitives that have proven stable across multiple bounded contexts.

Adopt: the Java language and established libraries rather than creating a DDD framework.

### 2.3 `outbox` — adopt an existing delivery mechanism

This is one of the modules where I would be especially careful about custom implementation.

### Option A: Debezium CDC

Best candidate if your product uses a supported database and Kafka Connect-compatible infrastructure.

- Business data and the outbox record are committed in one database transaction.
- Debezium captures the outbox table changes.
- Its Outbox Event Router transforms those changes into outgoing messages.
- Your platform standardizes the table convention, event contract, configuration and operational requirements.

[Official Outbox Event Router documentation](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html?utm_source=chatgpt.com)

### Option B: Application-managed JDBC outbox

Consider if you don't want to operate a CDC pipeline, or your infrastructure is better suited to polling.

- Use the existing database transaction management.
- Select a maintained outbox library if one meets your needs, or implement a narrowly scoped publisher.
- Define retry, claiming, concurrency, cleanup, observability and recovery behavior explicitly.

This option has more application-level implementation and operational responsibility than a ready-made CDC path.

Debezium's official design specifically captures outbox table changes and routes them to messaging destinations. It does not remove the need to define application transaction boundaries, event contracts, consumer idempotency, or delivery semantics.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://debezium.io\&sz=32)

Debezium

+1



My recommendation: do not build the polling scheduler, lease manager, retry engine and locking mechanism until you've evaluated Debezium and a maintained outbox library against your infrastructure.

Also, don't promise end-to-end exactly-once business processing merely because the outbox is transactional. Consumers still need appropriate duplicate-handling behavior.

### 2.4 `eventing` — use the broker's supported integration

If Kafka is your messaging backbone, use the Micronaut Kafka integration rather than implementing Kafka producer and consumer infrastructure yourself. Micronaut already supplies producer and consumer support.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://micronaut-projects.github.io\&sz=32)

micronaut-projects.github.io



Your platform can provide:

- Consistent event headers and correlation metadata.
- Standard serialization configuration.
- Producer and consumer defaults.
- Error handling and retry conventions.
- Health and operational metrics.
- A common event-publishing API only if it meaningfully simplifies application code.

Keep broker-specific functionality accessible where needed. Don't create a lowest-common-denominator abstraction that prevents services from using essential Kafka features.

If your product uses RabbitMQ, Pulsar, or another broker, the exact integration choice changes. Do not introduce a second broker just to make the platform appear generic.

### 2.5 `cqrs` — build conventions, not another framework

Your requirement is explicitly for lightweight CQRS.

Use ordinary Java command/query objects, handler classes and Micronaut dependency injection. Add a thin dispatcher only if you have a clear need for common validation, authorization, transaction handling, or middleware.

For example, the platform could standardize the handler contract while allowing each service to implement its own business logic.

Do not introduce Axon solely because it already implements CQRS. It is worth evaluating if you later need its broader capabilities, but it is not necessary for a lightweight command/query model.

Decision: build a small module only after agreeing on handler contracts and identifying the common behavior that must be standardized.

### 2.6 `observability` — integrate established standards

Use:

- Micronaut Tracing with OpenTelemetry for distributed tracing.
- Micrometer for application metrics.
- Prometheus-compatible metrics exposition where required.
- Structured logging with trace and correlation context.
- Your organization's existing audit logging infrastructure, if available.

Micronaut provides OpenTelemetry integration, including trace configuration and resource attributes. Its documentation also emphasizes keeping the relevant OpenTelemetry dependencies aligned with the Micronaut tracing BOM.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://micronaut-projects.github.io\&sz=32)

micronaut-projects.github.io

+1



Your platform should configure the defaults and make it easy for every service to comply. It should not invent a tracing protocol, metrics API or logging framework.

One important distinction: audit logging is not the same as ordinary application logging. Security-relevant audit events may need separate retention, access controls, integrity guarantees and delivery requirements.

### 2.7 `event-schema` — evaluate CloudEvents rather than inventing an envelope

Use the Java SDK for CloudEvents if its model fits your event architecture. It supports creating and manipulating event envelopes, serialization and supported protocol bindings.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://cloudevents.github.io\&sz=32)

Java SDK for CloudEvents

+1



Your platform should define:

- Required metadata and naming conventions.
- Schema versioning and compatibility policy.
- Payload serialization rules.
- Event ownership and evolution procedures.
- Schema registry integration, if your infrastructure requires one.

CloudEvents standardizes event metadata and representation; it does not automatically provide schema governance or guarantee compatibility between versions.

Decision: adopt an existing envelope standard where appropriate and own only the organization-specific rules.

### 2.8 `security` — don't create a custom authentication framework

Use Micronaut Security and your existing identity provider for authentication. Micronaut already supports JWT-based authentication.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://guides.micronaut.io\&sz=32)

Java / Gradle



Your platform can standardize:

- How tenant identity is established and propagated.
- How authenticated identities reach application use cases.
- How permission checks are invoked consistently.
- How audit identities are attached to relevant operations.
- How security configuration is validated across environments.

For centralized policy evaluation, evaluate a dedicated policy engine only if your requirements justify one. A simple application-level authorization check does not automatically need an additional service or policy language.

Do not build your own JWT validator, token cryptography, identity provider, or proprietary security protocol.

### 2.9 `search-projection` — adopt the official search client

If your search engine is Elasticsearch, use its official Java client. Its Bulk Ingester already provides batching based on operation count, payload size, and elapsed time, with configurable concurrency and backpressure.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.elastic.co\&sz=32)

Java



Your platform can supply:

- A projection handler convention.
- Standard error classification and retry policies.
- Rebuild and reprocessing procedures.
- Consistent index naming and mapping ownership rules.
- Observability for projection lag and failures.

You still need to design ordering, duplicate handling, deletion semantics, and projection rebuild behavior.

If you're using OpenSearch instead, evaluate its corresponding Java client and Micronaut integration. Don't introduce Elasticsearch-specific abstractions into the core platform API unless you intend to support only Elasticsearch.

### 2.10 `workflow` — use Temporal directly underneath a thin API

Adopt the Temporal Java SDK for durable workflow execution. It already supports workflows, activities, retries, timers, cancellation and workflow versioning.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.temporal.io\&sz=32)

Temporal Documentation



Your `workflow` module should provide a smaller, documented API for common organizational patterns, such as:

- Starting a named business workflow.
- Passing a typed workflow input.
- Handling workflow status and outcomes.
- Applying approved timeout and retry defaults.
- Exposing operational correlation information.

Keep advanced Temporal capabilities accessible to teams that need them. Do not build a custom durable workflow engine, scheduler or retry engine on top of Temporal.

### 2.11 `testing` — compose existing test libraries

Use:

- JUnit Jupiter for unit and integration tests.
- Testcontainers for dependencies such as databases, Kafka and search engines.
- Micronaut's test utilities for application-context and HTTP testing.
- Architecture tests to enforce module boundaries.
- Contract tests where services exchange events or APIs.

Testcontainers provides a way to run dependencies in containers for automated tests; it is a better starting point than building custom database and broker lifecycle management for your testing kit. See the [Testcontainers Java documentation](https://java.testcontainers.org/?utm_source=chatgpt.com).

Build shared fixtures, event test utilities, standard test configuration and helpers for common platform failure scenarios. Avoid making the testing module a second test framework.

### 2.12 `autoconfiguration` and starters — this is where your platform adds real value

These modules are good candidates for custom development because they connect the existing components in the way your organization wants to use them.

For example:

- `micronaut-platform-starter` includes only the foundational platform capabilities.
- `micronaut-outbox-starter` configures the selected outbox implementation.
- `micronaut-cqrs-starter` provides the agreed handler conventions.
- `micronaut-ddd-starter` establishes optional project structure and architecture checks.

Use conditional configuration so that a service does not need Temporal, search, or CQRS simply because it uses your platform.

Keep starters thin. Most of their job should be dependency selection, default configuration, and connecting existing implementations.

## 3. The architecture I would approve before implementation

Here is the intended relationship between the components. This is a proposed architecture, not a requirement to build every module immediately.

## Bounded-context services

Business logic, use cases, domain model and service-owned policies

Consume only required capabilities

### Your enterprise platform

Eventing / Outbox

Thin integration

Security

Shared policies

Observability

Defaults

CQRS / Workflow

Optional APIs

BOM, configuration, testing utilities and optional starters support the modules above.

Existing implementations and infrastructure

Micronaut

Application runtime

Database

Transactions and storage

Messaging / CDC

Event delivery

Telemetry / Temporal / Search

Specialized capabilities

The main architectural rule is that the platform depends on the existing technologies, not the other way around. Your domain model should not need to know whether an event is delivered through Debezium or whether a workflow runs on Temporal.

## 4. Decisions we must settle before coding

There are four important infrastructure choices that cannot be decided responsibly from the module diagram alone.

## Platform technology baseline

Select your current or intended choices. If something is undecided, choose that option. These answers help narrow the existing tools to evaluate before implementation.

1\. Primary relational database

PostgreSQL

MySQL

Oracle

Other / multiple databases

Not decided

2\. Messaging infrastructure

Apache Kafka

RabbitMQ

Cloud-managed messaging

Other / multiple brokers

Not decided

3\. Search engine

Elasticsearch

OpenSearch

Database search only for now

Other

Not decided

4\. Deployment environment

Kubernetes

Managed cloud containers / serverless

Virtual machines

Other

Not decided

5\. Durable workflow engine

Temporal is already selected

An existing workflow engine is in use

Simple application-level orchestration is enough

Not decided / compare options

Prepare technology decision record

## 5. What we should produce before the first implementation

I would make the next deliverable an architecture decision package containing:

1. Technology decision register: each module, selected existing tool, alternatives considered, reason for selection, and whether custom code is justified.
2. Dependency graph: which modules can depend on which other modules, with domain independence enforced.
3. Integration contracts: event envelope, outbox API, CQRS handler conventions, security context and workflow API.
4. Compatibility and licensing review: Java 21+, Gradle, Micronaut, selected integrations, transitive dependencies and licenses.
5. Operational responsibility matrix: who owns broker failures, outbox recovery, schema changes, security configuration, workflow recovery and dependency upgrades.
6. Implementation gates: measurable acceptance criteria before any module is considered ready for production.

My recommendation is to approve the technology choices and contracts first, then implement only the minimum platform slice required by one real bounded context. That will prevent us from writing code that simply duplicates what a mature component already does.
