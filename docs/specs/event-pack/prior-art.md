# Literature Review: Prior Art for a Declarative "Event Pack" Simulator in the Snowplow CLI

*A survey of existing synthetic / simulated event-generation tooling and the design patterns it establishes, as background for a proposed "event pack" feature in the Snowplow CLI. Tools are surveyed as publicly documented prior art; references to commercial or closed-source products are for design study only and imply no reuse of their implementations.*

*Terminology used throughout: an **event pack** is the YAML artifact (the declarative spec of events, journeys, and personas); **`events simulate`** is the proposed Snowplow CLI command that runs one. The feature would live in `snowplow-cli` as part of the `events` command family — alongside **`events track`**, the single-event command absorbed from the formerly-standalone `snowplow-tracking-cli` (see `docs/specs/` for the absorption spec).*

## Summary
- The closest analogues to the proposed "YAML event pack → simulated Snowplow events" feature are **Confluent's kafka-connect-datagen** (the schema-driven tool from the Kafka ecosystem) together with **ShadowTraffic** (the commercial successor to Voluble). Of these, ShadowTraffic is the most instructive reference point: its publicly documented design is the only mature example that combines declarative configuration, stateful/journey-aware generation (`stateMachine`), cross-entity referential integrity (`lookup`), and pluggable output sinks in a single config file.
- No surveyed tool performs declarative, journey-aware, Iglu / self-describing-JSON-schema-driven synthetic event generation emitted directly to a Snowplow collector — that specific combination appears to be genuinely novel. Snowplow already ships a relevant-but-different tool, **snowplow-event-generator** (Scala, HOCON-configured), which produces *enriched / atomic* events with duplicate and context distributions and can sink to a collector over HTTP — but it generates random, non-journey-aware events and is not driven by the user's own Iglu schemas.
- The most instructive design patterns identified across the literature — all general, widely-used approaches documented in public sources — are: (1) a **schema-driven field grammar** (datagen's `arg.properties` with `options` / `range` / `regex` / `iteration`); (2) **stateful user-journey modeling** via state machines and per-user "forks" (ShadowTraffic) or weighted scenario "flows" (Artillery); (3) **referential integrity across entities** via lookups (ShadowTraffic / Voluble); and (4) **generation directly from JSON Schema** (json-schema-faker / `jsf`), which maps naturally onto Snowplow's Iglu registry.

## Key Findings

### The landscape splits into four archetypes
1. **Schema-driven record generators** — you describe one record's shape; the tool emits N independent randomized records. Examples: kafka-connect-datagen, ksql-datagen, Synth, Mockaroo, Materialize `datagen`, json-schema-faker / `jsf`, Benthos / Bento `generate`. Cheap to build, weak at realism (independent records don't form sessions / funnels).
2. **Scenario / journey-driven generators** — you describe a *flow* or *state machine* that a simulated user walks through, producing correlated sequences. Examples: ShadowTraffic (`stateMachine` + `fork`), Artillery / k6 / Gatling / Locust (load-test scenarios), jafgen (agent-like simulation). Best at realism; this is what makes behavioral data look real.
3. **Cross-entity relational generators** — emphasis on referential integrity between datasets (orders reference customers). Examples: Voluble / MSK Data Generator (cross-topic references), ShadowTraffic (`lookup`), Synth (relational import).
4. **Product-analytics demo seeders** — bespoke scripts that hydrate a specific product's analytics with realistic usage. Examples: PostHog's `posthog-demo-3000` (HogFlix / hedgebox), Snowplow's own snowplow-event-generator, Segment / RudderStack test-event utilities.

A Snowplow event-pack feature would sit at the intersection of archetypes (2) and (4), and could draw on the declarative spec grammar established by (1) and the referential model established by (3).

### Snowplow's existing tooling — the current baseline
- **snowplow-cli `events track`** (the single-event command formerly shipped as the standalone `snowplow-tracking-cli` repo, now absorbed into this CLI as part of the `events` family — this desk research predated the absorption and originally framed the feature as an extension of the standalone tool): a Go binary using the Snowplow Golang Tracker. It sends exactly one event per invocation via `--collector`, `--appid`, `--method` (GET / POST, default GET), `--protocol` (default https), and either `--sdjson` (a full self-describing JSON) or `--schema` + `--json` (which it assembles into a self-describing JSON), plus `--ipaddress` and `--contexts` (default `[]`). There is no buffering — each event is sent as an individual payload. The original standalone repo was Apache-2.0, single-digit stargazers and forks, latest release 0.7.0 (Jan 9, 2023). This is a thin, single-shot tool — the proposed `events simulate` feature would be its journey-aware, batched sibling in the same `events` family.
- **snowplow-event-generator** (snowplow-incubator, Scala): the closest existing Snowplow analogue. Configured via a **HOCON** file; generates *enriched* events (collector payloads + TSV + JSON) deterministically from a seed, with knobs for `payloadsTotal`, `eventPerPayloadMin/Max`, `eventFrequencies` (struct / unstruct / pageView / pagePing and per-schema unstruct weights), `contexts.minPerEvent/maxPerEvent`, and a `duplicates` block (natural and synthetic duplicate probabilities and pool sizes). Output sinks: File / S3, Kafka, Kinesis, PubSub, EventHubs, and **HTTP (a Snowplow collector endpoint)**. Crucially it generates *random* events, not journey-aware sessions, and is not driven by the user's own Iglu schemas. Apache-2.0, latest 0.10.0 (Jan 2026), small community (single-digit stars / forks).
- **snowplow-cli** `ds generate`: generates a *schema template* (an empty data-structure scaffold), NOT sample data. **igluctl** `static generate` produces DDL / JSONPaths / migrations, NOT sample data. **Snowplow Micro** validates incoming events (good / bad endpoints) but does not synthesize them. **Net: nothing in the Snowplow / Iglu ecosystem today generates sample event *instances* from an Iglu JSON Schema.** This is the white space.
- **Tracker protocol the feature would emit to**: POST to `/com.snowplowanalytics.snowplow/tp2`, body = a self-describing JSON `iglu:com.snowplowanalytics.snowplow/payload_data/jsonschema/1-0-4` whose `data` is an **array of event maps** — so a "pack" can be sent as efficient batched POSTs. Key params: `e` (pv / pp / se / ue / tr / ti), `tv` (tracker version), `p` (platform), `aid` (app id), `eid` (UUID), `tna` (tracker namespace), `dtm` / `stm` / `ttm` timestamps, `ue_px` (base64 self-describing event wrapped in `iglu:com.snowplowanalytics.snowplow/unstruct_event/jsonschema/1-0-0`), and `co` / `cx` (contexts wrapped in `iglu:com.snowplowanalytics.snowplow/contexts/jsonschema/1-0-1`). Session / user fields like `duid`, `sid`, `vid`, `uid` are exactly what a journey model would hold as per-user state.

## Details

### 1. Kafka ecosystem (the Datagen family)

**Confluent kafka-connect-datagen / Datagen Source Connector.** Schema-driven. Core abstraction: an **Avro schema** (`.avsc`) annotated with `arg.properties` rules consumed by the **Avro Random Generator**. Bundled "quickstart" packs map a name (`users`, `pageviews`, `clickstream`, `clickstream_users`, `clickstream_codes`, `orders`, `ratings`, `stock_trades`, etc.) to a bundled schema. Config knobs: `quickstart` OR `schema.filename` / `schema.string`, `schema.keyfield`, `max.interval` (delay between records), `iterations` (total records). Output: Kafka topics, in Avro / JSON / JSON-SR / Protobuf. Independent records only — no journeys. Illustrative field grammar:
```json
{
  "name": "userid",
  "type": { "type": "string",
    "arg.properties": { "regex": "User_[1-9]" } }
},
{
  "name": "rating",
  "type": { "type": "int",
    "arg.properties": { "range": { "min": 1, "max": 5 } } }
},
{
  "name": "brand",
  "type": { "type": "string",
    "arg.properties": { "options": ["Acme", "Globex"] } }
}
```
`arg.properties` supports `options` (enumerated choices), `range` (min / max), `regex` (pattern), and `iteration` (a monotonically increasing counter with `start` / `step` — useful for sequential timestamps / IDs). This style of field grammar is a well-established, widely-used approach that a Snowplow event-pack field spec could follow. **ksql-datagen** is the older CLI form of the same engine: `ksql-datagen quickstart=pageviews topic=pageviews`, with `iterations` (default 1,000,000) and `maxInterval` (default 500ms).

**Voluble (MichaelDrogalis/voluble).** Clojure Kafka Connect connector, now **deprecated** in favor of ShadowTraffic — the repo carries an explicit banner noting that the author has launched an improved commercial offering (ShadowTraffic) to sustainably provide a similar service. (Author Michael Drogalis; his GitHub profile describes him as a product manager at Confluent and author of Onyx.) Core abstraction: directive-keyed properties of the form `<directive>.<topic>.[attribute].[qualifier].<generator> = <expression>`, where directives are `genk` / `genkp` / `genv` / `genvp` (generate key / key-primitive / value / value-primitive). Realism features that matter for design: **cross-topic references** (`genvp.publications.matching = users.key`), **Java Faker expressions** (`#{Name.full_name}`), distributions (`matching` with uniform / normal), bounded uniqueness (`within`), a `sometimes` qualifier with configurable `matching.rate`, throttling (`global.throttle.ms`, `topic.<t>.throttle.ms`), and **tombstoning** (null-valued keys). AWS re-implemented it in Java as **amazon-msk-data-generator**, whose documentation describes it as a Clojure-to-Java translation of Voluble and highlights the ability to generate events that reference other generated events. ~155 GitHub stars / ~11 forks, EPL license.

**ShadowTraffic (commercial successor to Voluble) — the most instructive reference point.** Its documentation describes it as a containerized service for declaratively generating data, with extensive controls for mimicking production traffic. JSON or YAML config with two top-level keys: `generators` (what to produce) and `connections` (where to send it). Among surveyed tools it is the only mature example whose public design combines all of the properties a Snowplow event-pack feature would need:
- **Field grammar** via `_gen` functions: `oneOf` / `weightedOneOf` (choices), `uniformDistribution` / `normalDistribution` (`bounds` / `mean` + `sd`), `uuid`, `now`, `string` with Java-Faker `expr` (`#{Internet.url}`, `#{Commerce.productName}`).
- **Stateful user journeys** via `stateMachine` (`initial`, `transitions`, `states`), where transitions can be probabilistic (`oneOf` over next states) and terminal states model funnel exits / abandonment. Each state overrides the base generator and can carry its own `throttleMs` / `delay`.
- **Per-user simulation** via `fork` (`key`, `maxForks`, `stagger.ms`) — spins up N concurrent "users," each running its own state-machine instance with a `forkKey` variable. `previousEvent` lets a value depend on the prior event (e.g., a running cart total).
- **Referential integrity** via `lookup` across connections / topics (e.g., `tweets.userId` looks up IDs already written to `users`).
- **Scheduling** via top-level `schedule.stages` (run generators in sequence for seeding) with per-stage `overrides`.
- **Emission**: Kafka, Postgres, file system (json / jsonl / parquet), with `--stdout`, `--sample N`, `--watch`, and `--seed` for repeatable runs. It also has `--bootstrap-from-json-schema` to scaffold config from a JSON Schema.

Illustrative journey / funnel spec (reconstructed from ShadowTraffic's public documentation to show the *shape* of a state-machine config — the closest analogue to a Snowplow "event pack"):
```json
{
  "topic": "funnelEvents",
  "fork": { "key": {"_gen":"string","expr":"#{Name.username}"},
            "stagger": {"ms": 200} },
  "key": {"_gen":"var","var":"forkKey"},
  "stateMachine": {
    "_gen": "stateMachine",
    "initial": "viewLandingPage",
    "transitions": {
      "viewLandingPage": "addItemToCart",
      "addItemToCart": {"_gen":"oneOf","choices":["viewCart","addItemToCart"]},
      "viewCart": "checkout"
    },
    "states": {
      "viewLandingPage": {"value":{"stageName":"landingPage","referrer":{"_gen":"string","expr":"#{Internet.url}"}}},
      "addItemToCart": {"value":{"stageName":"addItem","item":{"_gen":"string","expr":"#{Commerce.productName}"}}},
      "viewCart": {"value":{"stageName":"checkCart","timestamp":{"_gen":"now"}}},
      "checkout": {"value":{"stageName":"purchase","price":{"_gen":"uniformDistribution","bounds":[1,100]}}}
    }
  },
  "localConfigs": {"throttleMs": 800}
}
```
ShadowTraffic is closed-source commercial software (the free tier requires a license key). It is referenced here purely as publicly documented prior art for design study; the concepts it surfaces (state machines, per-user forks, cross-entity lookups) are general and appear across the field, and none of its implementation would be reused.

**Materialize `datagen`** is a lighter open-source CLI worth noting: it takes JSON / Avro / SQL schemas using **FakerJS** expressions (`faker.internet.userName()`), supports a `_meta` block with `topic`, `key`, and **`relationships`** (parent / child field joins for referential integrity), and emits to Kafka (JSON / Avro) or Postgres with `-n` record count and `-w` wait between records. Permissively licensed, and a closer code model for a CLI than the JVM connectors.

### 2. General declarative synthetic-data libraries

**Synth (getsynth / synth, now under shuttle-hq).** A declarative data generator. Rust CLI. Core abstraction: a **namespace** (directory) of **collection** JSON files; each collection is a generator tree of typed nodes (`array`, `object`, `number`, `string`, `date_time`) where leaves can call **Faker** generators (`"faker": {"generator": "safe_email"}`) or special types (`"id": {}` for auto-increment). Supports relational `import` from Postgres / MySQL / MongoDB (infers PK / FK) and `generate --to postgres://...`. Emits batch JSON to stdout or databases; not streaming, no journeys. Apache-2.0. Illustrative:
```json
{ "type": "array", "length": {"type":"number","constant":1},
  "content": { "type":"object",
    "id": {"type":"number","id":{}},
    "email": {"type":"string","faker":{"generator":"safe_email"}},
    "joined_on": {"type":"date_time","format":"%Y-%m-%d","subtype":"naive_date","begin":"2010-01-01","end":"2020-01-01"} } }
```

**Faker / Faker.js / Python Faker** — the underlying primitive nearly every tool builds on (names, emails, addresses, commerce, internet). Not declarative itself; it is the value-generation library a tool embeds. A Go equivalent (e.g., `brianvoe/gofakeit` or `jaswdr/faker`) is the natural building block given the CLI is Go.

**Mockaroo.** Schema-driven, GUI + REST API. ~74 field types, `Custom List`, formulas, conditional `if` logic, dataset references for referential integrity. API: POST `/api/generate.(json|csv|...)` with a saved schema name or an inline `fields` array (`{name, type, values/min/max/format}`). Batch only (CSV / JSON / SQL / Excel). Commercial with a free tier; surveyed as prior art only.

**Benthos / Bento (warpstreamlabs/bento) and Redpanda Connect.** The `generate` input emits messages on an `interval` (duration or cron, e.g., `@every 5m`), with `count` and `batch_size`, each built by a **Bloblang** `mapping`. Bloblang is a full mapping DSL (`root.id = uuid_v4()`), so realism is hand-coded rather than schema-declared. Output to anything (200+ connectors incl. Kafka / HTTP / stdout). MIT / Apache. A useful model for "declarative interval + mapping → stream," but no first-class journey or schema concept.

**json-schema-faker (JS) and `jsf` (Python, ghandic/jsf).** Directly relevant to Snowplow's Iglu setup: both **generate fake data instances *from* a JSON Schema**. json-schema-faker supports Draft 2019-09 / 2020-12, `$ref` with cycle detection, composition (`allOf` / `anyOf` / `oneOf`), seeded determinism, and Faker / Chance hints (`"faker": "person.fullName"`). Python `jsf` uses `"$provider": "faker.name"` and supports **multi-level state for dependent data** (e.g., children sharing a surname). Both have CLIs and are open source. **This is a proven, openly licensed mechanism for the Iglu-schema-driven half of the feature** — point it at an Iglu schema's `properties` and you get a valid event body for free. (Caveat: an Iglu schema is a self-describing JSON Schema, so generation runs against `properties`, then re-wraps in the self-describing envelope.)

**jafgen (dbt-labs/jaffle-shop-generator).** Python CLI; `jafgen <years>` simulates a fictional café (customers, stores, products, orders) with **agent-like behavior and seasonality** (weekends slower, customer preferences, new stores opening over time), emitting CSVs for dbt seeds. Explicitly **not idempotent** — it generates new data on each run based on the simulation's interactions. Apache-2.0. A good reference for "simulation produces a coherent dataset" rather than "schema produces random rows." There is even a Singer tap (`tap-jaffle-shop`) wrapping it.

**Gretel.ai** (for contrast): ML / generative synthetic data trained on real data to preserve statistical properties / privacy — the wrong paradigm for seed / demo event packs, which need declarative control rather than learned distributions.

### 3. Analytics / product-event simulators

- **PostHog `posthog-demo-3000`** (HogFlix demo app): a Python `seed_demo_data.py` script that creates pseudo-random data mimicking real usage, seeded from a `500_names_and_emails.csv` of 500 users with group / family info, posting historic events into a PostHog project via the API. A bespoke simulation script, not a declarative spec — but it confirms the product-analytics need for *journey-shaped* demo data (funnels, retention, paths) rather than random events.
- **Segment**: "Generate sample event" in the Event Tester / Mappings Tester produces a single synthetic Track / Identify / Page / Screen / Group payload for testing a destination mapping — single-event, not a pack. Segment's Protocols Tracking Plans validate events against JSON Schema (analogous to Iglu).
- **RudderStack**: an **Event Playground** app for sending sample events without instrumentation, plus a `./scripts/generate-event <writeKey> <dataPlaneURL>/v1/batch` shell script in rudder-server. Also notable: RudderStack's **Data Catalog uses YAML** (`kind: events`, `spec.events: [{id, name, event_type, description, category}]`) to define tracking plans — a precedent for YAML event definitions in this space, though for governance rather than generation.
- **Amplitude / mParticle**: ship demo datasets / sample data but no open declarative generator surfaced.

### 4. Scenario / user-journey modeling (the most important design dimension)

The central design choice is **independent random records vs. stateful journeys**. Independent records (datagen, Synth, Mockaroo, json-schema-faker) are trivial to build but produce behaviorally meaningless data — no sessions, no funnels, no retention curves. Snowplow is a *behavioral* data platform, so the value of an event-pack feature hinges on journey awareness. Prior art for journeys:
- **ShadowTraffic `stateMachine` + `fork`** (above) — the most complete published model: probabilistic transitions, terminal states for abandonment, per-user forks with staggered arrival, `previousEvent` dependencies.
- **Load-testing scenario DSLs** — Artillery, k6, Gatling, Locust. **Artillery** is the cleanest YAML prior art: a `config` block (`target`, `phases` with `duration` / `arrivalRate` / `rampTo` / `arrivalCount`) plus `scenarios`, each a `flow` of steps with `think` (pauses), `loop`, `capture` (extract a value from one response to use in the next — referential chaining), `weight` (relative scenario frequency), and `expect`. Illustrative Artillery journey:
```yaml
config:
  target: "https://shop.example.com"
  phases:
    - duration: 300
      arrivalRate: 10
scenarios:
  - name: "Shopping flow"
    weight: 7
    flow:
      - get: { url: "/api/products" }
      - think: 3
      - post:
          url: "/api/cart"
          json: { productId: "{{ productId }}", quantity: 1 }
  - name: "Quick browse"
    weight: 3
    flow:
      - get: { url: "/api/products" }
      - think: 5
```
The Artillery model — **arrival phases** (shape of traffic over time) + **weighted scenarios** (which journeys, how often) + **flows with think-time and value capture** — maps almost one-to-one onto what a Snowplow event pack needs: emission rate over time, mix of user personas, and ordered events with session continuity.
- **Markov-chain / agent-based** approaches (e.g., Markov models for packet timing in network traffic tooling; jafgen's agents) are the academic end of the spectrum; ShadowTraffic's `stateMachine` is the pragmatic, declarative middle ground.

### 5. Common declarative-spec design vocabulary (synthesis)

Across the surveyed tools, a mature spec converges on these dimensions — a synthesis of conventions common across the field, and a useful starting vocabulary for a YAML event pack:
- **Field types & value sources**: primitive types, enumerated `options` / `oneOf`, `range` / distributions (uniform, normal, weighted), `regex` / patterns, Faker providers, `iteration` / auto-increment, `uuid`, `now` / time functions.
- **Distributions & weighting**: per-field statistical distributions; per-scenario / persona weights.
- **Relationships / referential integrity**: cross-entity `lookup` / `matching` / `relationships` so a `purchase` references a real `user` / `product`.
- **Cardinality**: number of distinct users / sessions (`maxForks`), records per entity, events per session (`eventPerPayloadMin/Max`).
- **Temporal behavior**: time ranges (`begin` / `end`), emission `interval` / `maxInterval` / `throttleMs`, arrival `phases` / `rampTo`, `think` / `delay`, `stagger`.
- **Sequencing / state**: ordered `flow` or `stateMachine` transitions, terminal / abandonment states, dependence on `previousEvent`.
- **Volume & control**: total `iterations` / `count` / `payloadsTotal`, `seed` for determinism, duplicate injection.
- **Output**: sink type (stdout / file / HTTP / stream), batch vs stream, format.

## Design Implications for a Snowplow Event-Pack Feature

**Stage 1 — A schema-driven, batch "pack" MVP (informed by datagen and json-schema-faker).**
Define a YAML pack with an event / field grammar following the well-established field-annotation style seen in datagen's `arg.properties` (`options`, `range`, `regex`, `iteration`) and, critically, support **generating event bodies directly from an Iglu schema** by fetching it from the user's Iglu registry and running json-schema-faker–style generation against its `properties` (using an open-source Go Faker such as `gofakeit`). Emit batched POSTs to the collector using the existing tracker (`payload_data/jsonschema/1-0-4`, array of event maps). This alone would be novel — nothing in the Snowplow / Iglu ecosystem generates sample event instances from Iglu schemas today. Milestone to move on: users can replay a real data structure as N synthetic events in one command.

**Stage 2 — Journey / session awareness (informed by the state-machine model in ShadowTraffic and the scenario model in Artillery).**
Introduce a `scenarios` (or `journeys`) section: a state machine of event types with probabilistic transitions and terminal (abandonment) states, plus `users` / `fork` cardinality with per-user session state (`duid`, `sid`, `vid`, `uid` held constant across a journey, `dtm` advancing with think-time). Add `phases` for arrival shaping and `weight` for persona mix. This is the feature's real differentiator and aligns with Snowplow's behavioral-data positioning. Milestone: generated data produces a realistic funnel / retention curve in Snowplow Micro or a warehouse, not just valid-but-random events.

**Stage 3 — Referential integrity & curated packs.**
Add `lookup`-style references so ecommerce `transaction` / `transaction_item` events reference generated products / users, and ship **bundled "quickstart" packs** (in the spirit of datagen quickstarts and PostHog HogFlix) — e.g., `ecommerce`, `media`, `saas-onboarding` — that work out of the box against Snowplow's standard schemas. Provide `--seed` for deterministic / repeatable runs (as snowplow-event-generator and ShadowTraffic both do) and `--dry-run` / `--stdout` for inspection (as ShadowTraffic and Materialize datagen do).

**Design patterns to adopt vs. avoid.**
- Adopt (these are general, publicly documented design patterns, not implementations): the compact field-annotation style seen in datagen; the state-machine + fork + lookup pattern documented by ShadowTraffic; Artillery's `phases` / `think` / `weight` / `capture` model; json-schema-faker's seedable JSON-Schema generation (open source); and snowplow-event-generator's own `seed`, `duplicates`, and HTTP-to-collector approach.
- Avoid: a hand-coded mapping DSL as the *only* interface (as in Benthos / Bloblang) — too low-level for the goal of saving users the trouble of hand-generating events; and ML-based generation (Gretel) — the wrong paradigm for declarative seed data.
- Coordinate, don't duplicate: position the CLI feature as the **lightweight, journey-aware, tracker-protocol** counterpart to the heavier Scala **snowplow-event-generator** (which targets enriched / atomic output and pipeline load testing). Consider whether the two should share a pack format.

**What would change these implications.** If user research shows the primary need is *pipeline load / scale testing* rather than *realistic demo / onboarding data*, extending snowplow-event-generator (built for volume and duplicates) may be preferable to the CLI. If the primary need is *single-event validation*, Snowplow Micro plus the existing one-shot CLI may already suffice and a full pack feature may be unwarranted.

## Limitations and Caveats
- **ShadowTraffic is closed-source commercial software** (a license key is required even for the free tier); it is treated throughout this review as a design reference only, never as a dependency or a source of code. **Voluble is deprecated.** **Mockaroo** and **Gretel** are commercial. Their general design concepts are surveyed as prior art; their implementations are not reused.
- **Maturity / popularity signals are modest** for the most directly comparable tools: snowplow-tracking-cli and snowplow-event-generator both have single-digit GitHub stars; Voluble has roughly 155. These are niche tools, and popularity should not be over-weighted. (GitHub star / fork counts fluctuate and differ slightly between cached and live views; treat them as order-of-magnitude.)
- **Snowplow doc paths have been reorganized** (e.g., `event-studio` vs `data-product-studio`); both currently serve identical snowplow-cli content. Schema versions drift: the `contexts` wrapper appears as both `1-0-0` and `1-0-1`, and `payload_data` has incremented over time (older trackers reference `1-0-3`); `1-0-4` is current per the docs. Exact versions should be confirmed against the live Iglu Central at build time rather than hard-coded.
- The observation that "nothing does declarative, journey-aware, Iglu-schema-driven generation to a collector" is based on a thorough but not exhaustive survey; a community or internal tool may exist. The combination, however, is clearly not addressed by any mainstream tool found.
- Several tools' realism is only as good as their configuration — datagen / Synth / Mockaroo produce *independent* records by default, which look valid but are behaviorally meaningless; this is the trap a Snowplow event-pack feature should avoid by making journeys first-class.
