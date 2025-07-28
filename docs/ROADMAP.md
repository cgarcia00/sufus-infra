# Sprint Plan: Real‑Time Notifications Hub (with Generative Summaries)

> **Scope:** Serverless Java Lambdas with Dagger, TypeScript CDK, React (S3 + CloudFront), Cognito auth, DynamoDB single-table, Bedrock for summarization. Initial sources: Generic Webhook + GitHub. Micro‑summary window: 5 minutes. Channels: WebSocket (real‑time) + Email (daily).

---

## Sprint 1 — Foundations & Infrastructure (Week 1)

### Goals

- Polyrepo scaffolding, CDK stacks, baseline Lambdas with Dagger, Cognito auth, Dynamo schema in place, CI skeleton.

### Key patterns to introduce

- **Repository:** abstract Dynamo access behind interfaces (usable in tests).
- **Facade:** wrap AWS SDK clients (`DynamoDbFacade`, `BedrockFacade`, `SesFacade`) to keep service code clean.
- **Builder (lightweight):** for CDK props and Java config objects.
- **Dependency Injection (Dagger):** top‑level `AppComponent` for singletons and per‑request subcomponents.

### Tasks

#### Repos & CI

- Create repos: `infra/`, `service-ingest/`, `service-preprocess/`, `service-summarize/`, `service-delivery/`, `service-websocket/`, `web-console/`, `shared-java/`.
- GitHub Actions: Java build/test; CDK synth on PR; main branch deploy to `dev` via GitHub OIDC role.

#### CDK stacks

- **CoreStack:** DynamoDB `Notifications` table (on‑demand), SQS (`eventsQueue`, DLQ), KMS key.
- **ApiStack:** API Gateway (REST `/events` + placeholder), API Gateway (WebSocket `/live` placeholder), Cognito User Pool & Identity Pool, permissions.
- **ObservabilityStack:** CloudWatch dashboard & basic alarms (Lambda errors, DLQ depth).

#### Java scaffolding

- `shared-java`: domain models (`NotificationEvent`, `Summary`, `UserPreferences`), `Repository` interfaces, DTOs.
- Dagger `AppComponent` with modules: `AwsModule`, `RepoModule`, `ConfigModule`, `ObjectMapperModule`.
- Example Lambda (“hello”) wired to Dagger to verify cold start wiring.

### Acceptance criteria

- `cdk deploy` stands up stacks in `dev`.
- A health Lambda reachable at `GET /health` returns **200**.
- Unit test proving `NotificationRepo.save()` persists and reads back a stub item via LocalStack/Testcontainers.

### Tests

- Infra snapshot tests (CDK assertions).
- Repo round‑trip with LocalStack.

---

## Sprint 2 — Ingestion & Normalization (Week 2)

### Goals

- Accept events from a **generic webhook** and **GitHub** webhook, normalize, persist, and enqueue for processing.

### Key patterns

- **Adapter:** `SourceAdapter` interface with implementations `GenericWebhookAdapter`, `GithubAdapter`.  
  **How:** Dagger multibinding `@IntoMap String -> SourceAdapter` and select by `sourceType`.
- **Repository + Unit of Work:** batch write normalized events & enqueue SQS message as one logical unit (idempotency key).
- **Decorator:** add logging/metrics to adapters without touching core logic.
- **Factory Method:** create `NormalizedEvent` from raw payload type safely.

### Tasks

#### API endpoints

- `POST /events` (generic) with shared secret header.
- `POST /sources/github/webhook` validating GitHub signature.

#### `service‑ingest` Lambda

- Parse payload → select `SourceAdapter` → produce `NotificationEvent`.
- Compute deterministic `hash` for dedupe.
- Write to Dynamo (`EVENT#<ts>#<eventId>`), publish minimal message to SQS (`userId`, `eventId`, `windowKey`).

#### Dagger wiring

- `SourceAdaptersModule` with map bindings; `LoggingAdapterDecorator` via Dagger to wrap adapters.

#### Idempotency

- Conditional writes in Dynamo on `hash` attribute; ignore duplicates.

### Acceptance criteria

- Post a sample GitHub PR event → see one `EVENT#…` item in Dynamo and one SQS message.
- Duplicate delivery does **not** create duplicates.
- Unit tests for adapters with recorded GitHub fixtures.

### Tests

- Contract tests: replay GitHub webhook JSONs.
- Idempotency test: send same payload twice; table count unchanged.

---

## Sprint 3 — Preprocessing Pipeline & Windowing (Week 3)

### Goals

- Group events per user into 5‑minute buckets, dedupe, redact, and prepare content for LLM.

### Key patterns

- **Chain of Responsibility:** pipeline of `PipelineStep`s (`DeduplicateStep`, `RedactStep`, `ClassifyStep`, `ClusterStep`, `ClampStep`).  
  **How:** Define `interface PipelineStep { Context apply(Context c); }` and register a `List<PipelineStep>` via Dagger multibinding; execute sequentially.
- **Strategy:** `GroupingPolicy` (start with time‑based `TimeWindowPolicy`).
- **Builder:** `PromptInputBuilder` (collects compact bullet inputs per event).
- **Decorator:** metrics/timing and error‑count wrappers for each step.

### Tasks

#### EventBridge schedule

- Rule `cron(*/5 * * * ? *)` firing a `SummarizeTick` Lambda or Step (pull users with active windows).

#### `service‑preprocess` Lambda

- For a given `(userId, windowKey)`, load events from `Notifications` table range.
- Run CoR pipeline; produce `PreparedWindow` (compact list: title, key facts, link).
- Persist a `WINDOW#…` interim item (optional) or pass to summarize via SQS.

#### Redaction

- Simple regex masks for tokens/PII patterns and repo secrets; make it a pluggable `Redactor` (**Strategy**).

### Acceptance criteria

- Ingest ≥5 mixed events; at tick time a `PreparedWindow` is produced and stored (or enqueued).
- Logs show each pipeline step executed; timing metrics in CloudWatch.
- Redaction removes test secrets from bodies.

### Tests

- Unit tests for each `PipelineStep`.
- Performance: pipeline completes `< 500 ms` for 30 events.

---

## Sprint 4 — Summarization & JSON Contract (Week 4)

### Goals

- Call Bedrock to generate **structured JSON** micro‑summaries; validate/repair; store `SUMMARY#…`; emit `summary_created`.

### Key patterns

- **Strategy:** `ModelSelector` chooses `micro` vs `daily`.
- **Builder:** `PromptBuilder` creates system/user prompts with strict JSON schema rules.
- **Decorator:** wrap Bedrock client with retry/backoff, token accounting, and caching (idempotency by `windowKey`).
- **Observer:** publish `SummaryCreated` domain event (internal) on success.
- **Facade:** `BedrockFacade` hides SDK details and content filtering.
- **State (light):** `SummaryStatus` transitions (`PENDING -> READY -> FAILED`).

### Tasks

#### `service‑summarize` Lambda

- Receive `(userId, windowKey)`; load `PreparedWindow`.
- Build prompt; call Bedrock small model; parse JSON to `Summary`.
- Validate against a JSON schema; if invalid, attempt one **repair pass** (ask model to fix to schema).
- Store `SUMMARY#<windowStart>#<windowEnd>` with `includedEventIds`.
- Publish `SummaryCreated` to an internal SNS topic or invoke delivery directly.

#### Error handling

- DLQ for Bedrock failures; structured error codes.

### Acceptance criteria

- For a real window, Lambda writes a valid `Summary` item with `headline` and 2–6 `bullets`.
- JSON schema validator passes; if Bedrock returns invalid JSON, repair flow kicks in (log metric).
- Unit tests for `PromptBuilder` (deterministic inputs), schema validator, and error paths.

### Tests

- Snapshot test: same `PreparedWindow` → same prompt text (deterministic trimming).
- Cost guardrail: assert token estimate under threshold.

---

## Sprint 5 — Delivery (WebSocket & Email) + Preferences (Week 5)

### Goals

- Push summaries in real time over WebSocket; send scheduled daily emails; add user preferences.

### Key patterns

- **Strategy:** `DeliveryChannel` interface with `EmailChannel`, `WebSocketChannel`.
- **Observer:** on `SummaryCreated`, notify configured channels.
- **State:** delivery record transitions (`PENDING → SENT → ACKED/FAILED`) per channel.
- **Template Method:** base class `AbstractEmailFormatter` with `renderHeader/body/links`.
- **Proxy (optional):** WS client proxy wrapping API Gateway Management API.

### Tasks

#### `service‑delivery` Lambda

- Listen to `SummaryCreated` (SNS or direct call).
- Check `UserPreferences` (`PREF#`) for channels and quiet hours.
- Deliver via strategy; write `DELIV#<summaryId>#<channel>` records with status.

#### `service‑websocket` Lambda

- Handle `$connect` (auth via Cognito), `$disconnect`, and post to connections.

#### Preferences API

- `PUT /preferences` with fields: `{ microWindowMinutes, verbosity, channels, quietHours }`.
- Persist under `PREF#` item.

### Acceptance criteria

- Browser connected to `/live` receives `summary_created` event on generate.
- Quiet hours suppress email delivery but still store summary.
- SES email arrives with headline, bullets, and links; bounces/retries handled.

### Tests

- WS integration test: connect, simulate `SummaryCreated`, expect message.
- Email formatter unit tests (pure string checks).
- Preference‑driven routing unit tests.

---

## Sprint 6 — React Console, UX Polish & Guardrails (Week 6)

### Goals

- Usable console: live feed, daily digest page, preferences, source connection view; add costs/latency dashboards and budgets; micro‑to‑daily hierarchical flow.

### Key patterns

- **Composite** (front‑end data model): group bullets by topic for display (mirrors backend clustering).
- **Mediator** (light in UI): a store/controller that coordinates WS updates, fetches, and toasts.

### Tasks

#### `web‑console`

- Cognito Hosted UI sign‑in.
- **Live Feed:** subscribe WS; list summaries; expand shows bullets + “view underlying events”.
- **Daily Digest:** page rendering `granularity=daily` summaries.
- **Preferences:** form with validation; call backend.
- **Sources:** show webhook URLs & secrets (generic + GitHub).

#### CDK guardrails

- AWS Budgets alarms on Bedrock usage; CloudWatch dashboard (LLM latency, DLQ depth, WS connections).

#### Daily digest job

- EventBridge `cron(0 18 * * ? *)` → summarize the day from **micro‑summaries** (hierarchical summarization).

### Acceptance criteria

- Login → live updates appear; clicking a bullet opens original links.
- Preferences change affects next window.
- Budgets alarm configured; dashboard shows non‑zero metrics during tests.
- Daily digest email lands at **18:00 PT** with rolled‑up bullets.

### Tests

- Cypress (or Playwright) smoke test for auth + live message.
- Contract test that daily uses micro‑summaries (not raw events).

---

## Backlog (ready‑to‑pick tickets per sprint)

> Use/rename as GitHub Issues.

### Sprint 1

- `infra`: Create Notifications Dynamo table (PK/SK), GSIs.
- `shared-java`: Define domain models and `Repository` interfaces. **(Repository)**
- `service-common`: Implement `DynamoNotificationRepo`. **(Repository + Facade)**
- `infra`: REST & WS APIs; Cognito pools.
- `ci`: GitHub OIDC deploy role; CDK synth workflow.

### Sprint 2

- `service-ingest`: Implement `SourceAdapter` SPI + Dagger map. **(Adapter + Decorator)**
- `service-ingest`: Generic webhook endpoint with HMAC validation.
- `service-ingest`: GitHub webhook signature verification & adapter.
- `service-ingest`: Idempotent write (conditional put on `hash`).
- `infra`: Wire SQS + DLQ; send message per event.

### Sprint 3

- `service-preprocess`: Implement `PipelineStep` chain. **(Chain of Responsibility)**
- `service-preprocess`: `GroupingPolicy` time‑based. **(Strategy)**
- `service-preprocess`: `Redactor` SPI with default regex redactor. **(Strategy)**
- `service-preprocess`: Clamp & canonicalize event text.
- `infra`: EventBridge `*/5` rule; invoke preprocess/summarize flow.

### Sprint 4

- `service-summarize`: `PromptBuilder` with JSON schema. **(Builder)**
- `service-summarize`: `ModelSelector` micro vs daily. **(Strategy)**
- `service-summarize`: `BedrockFacade` + retry/backoff. **(Facade + Decorator)**
- `service-summarize`: Persist `SUMMARY#…` and publish `SummaryCreated`. **(Observer)**
- `service-summarize`: Repair loop for invalid JSON.

### Sprint 5

- `service-delivery`: `DeliveryChannel` SPI; implement WebSocket + SES. **(Strategy)**
- `service-delivery`: Delivery records with status. **(State)**
- `service-websocket`: `$connect/$disconnect` + post API proxy. **(Proxy, optional)**
- `api`: `PUT /preferences` and repo.
- `infra`: SNS (optional) or direct invoke from summarize to delivery.

### Sprint 6

- `web-console`: Auth, live feed, digest page, preferences.
- `infra`: Budgets, dashboards.
- `service-summarize`: Daily digest (hierarchical summarization).
- `service-preprocess`: Topic keyword grouping (seed for future semantic clustering). **(Strategy/Composite)**

---

## Dagger Wiring (summary checklist)

- `AppComponent` (Singletons): `AwsClients`, `Config`, `ObjectMapper`, `NotificationRepo`, `SummaryRepo`, `PreferencesRepo`, `BedrockFacade`, `Clock`.

**Multibindings**

- `Map<String, SourceAdapter>` → `"generic"`, `"github"`. **(Adapter)**
- `List<PipelineStep>` → `dedupe`, `redact`, `classify`, `clamp`. **(Chain)**
- `Map<String, DeliveryChannel>` → `"ws"`, `"email"`. **(Strategy)**
- `GroupingPolicy` bound to `TimeWindowPolicy`. **(Strategy)**
- Optional `Redactor` implementations; pick with qualifier.

**Subcomponents per Lambda**

- `IngestRequestComponent(rawPayload, headers)`
- `WindowPreprocessComponent(userId, windowKey)`
- `SummarizeComponent(userId, windowKey, granularity)`
- `DeliverComponent(summaryId, channel)`

---

## Definition of Done (global)

- **Reliability:** DLQs empty after load test; p95 Lambda `< 1s` (excl. Bedrock call).
- **Security:** All secrets in Secrets Manager/SSM; no secrets in logs; HMAC checks for webhooks; Cognito‑authenticated WebSocket connections.
- **Cost:** Budget alarm configured; micro summary token budget `<` configured ceiling.
- **Observability:** Correlation IDs in logs; dashboard with LLM latency, SQS age, DLQ depth, WS connections; alarms wired.
- **Docs:** README per repo; runbooks for replaying DLQ, rotating secrets, and deploying.

---

## How the GoF patterns map (cheat sheet)

- **Adapter** → Source connectors (GitHub, generic); lets you add Jira/Slack later without touching ingest flow.
- **Strategy** → Grouping policy, redaction policy, model selection, and delivery channels.
- **Chain of Responsibility** → Preprocess pipeline for events.
- **Builder** → Prompt construction and “prepared window” payloads.
- **Decorator** → Cross‑cutting: logging, metrics, retries, caching around adapters and Bedrock calls.
- **Observer** → `summary_created` notification to delivery.
- **State** → Delivery record lifecycle (`pending/sent/acked/failed`).
- **Repository (+ Unit of Work)** → All Dynamo access behind interfaces.
- **Facade** → AWS SDK wrappers (Dynamo, Bedrock, SES, WS Management).
- **Proxy** → Optional wrapper for API Gateway Management API used by WebSocket delivery.
