# Architectural Decisions

Each entry records a decision, its rationale, the alternatives considered, and any known trade-offs.

---

## ADR-001: Collapse domain/usecases and application/usecases into a single layer

**Decision:** Use case interfaces and their concrete implementations live together under `domain/usecases/`. There is no separate `application/` layer.

**Rationale:** The challenge explicitly asks for clean architecture and explicit DI, not strict Clean Architecture orthodoxy. A separate application layer would add directory depth and indirection without meaningful benefit at this project size. The key principle — use cases depend on repository interfaces, never on concrete implementations — is fully preserved.

**Trade-off:** In a larger codebase, collapsing these layers would make it harder to separate "what the system does" (use case interfaces) from "how it does it" (orchestration logic). Acceptable here; worth revisiting if the project grows.

---

## ADR-002: Factory functions + provide/inject for dependency injection

**Decision:** All concrete dependencies are instantiated in `plugins/dependencies.ts` via factory functions and registered with `app.provide()`. Composables retrieve them with `inject()` using typed `InjectionKey`.

**Rationale:**
- Idiomatic Vue DI — no external container library needed
- Composition root is explicit and in one place
- Composables are fully testable: mount with `app.provide(key, mockImpl)` before the test
- Demonstrates understanding of DI as a principle, not just a pattern

**Alternatives considered:**
- Direct singleton imports: simpler, but requires `vi.mock()` for every test and couples composables to concrete classes
- A DI container library (tsyringe, inversify): overkill for this size, adds decorator/reflect-metadata complexity

**Trade-off:** `inject()` can return `undefined` if called outside a component or composable context. Mitigated by writing a typed wrapper that throws a descriptive error if the key is not found — fail fast during development.

---

## ADR-003: WeatherUnit as flat union, validated by allowedUnitsByMetric map

**Decision:** `WeatherUnit` is a flat union of all possible unit strings. The relationship between metric and valid units is enforced at runtime via `allowedUnitsByMetric`, not via TypeScript generics.

**Rationale:** A generic `UnitForMetric<M extends WeatherMetric>` provides compile-time safety but is incompatible with Vue form state, which is always `string` at runtime. The form's `v-model` for the unit selector produces a `string`; casting it to a generic type at every boundary is noisy and error-prone.

**Trade-off:** The type system cannot catch an invalid metric/unit combination at compile time. This is accepted because `validateWeatherQuery()` catches it at runtime before the query reaches the use case, and the form dynamically renders only valid unit options from `allowedUnitsByMetric`.

---

## ADR-004: Daily aggregation produces a single value per day

**Decision:** `WeatherReading.value` is always a single number. For daily interval, the use case computes one representative value per day using the method defined in `dailyAggregationByMetric`.

**Rationale:** The chart and table both need a scalar value per timestamp. Exposing `maxValue`/`minValue` on `WeatherReading` would leak API structure into the domain and complicate every downstream consumer.

**Alternative considered:** Add `secondaryValue?: number` to support max/min range display. Rejected because it requires special-casing in both chart and table, and the chart library (Chart.js or similar) would need explicit range-band configuration. Out of scope for this challenge; can be added later.

**Trade-off:** The daily temperature value (mean of max/min) is an approximation. This is communicated to the user via a UI disclaimer derived from `dailyAggregationByMetric`. The domain carries the label; the component renders it.

---

## ADR-005: DataKind is a domain concept, not an infrastructure detail

**Decision:** `WeatherReading.kind: DataKind` ('historical' | 'forecast') is determined by comparing `timestamp` to `startOfToday`, regardless of which API endpoint the data came from.

**Rationale:** The visual distinction between historical and forecast data is meaningful to the user and belongs in the domain model. Coupling it to which endpoint was called would leak infrastructure structure into the UI.

**Rule:** `timestamp < startOfToday → 'historical'`, `timestamp >= startOfToday → 'forecast'`. Applied in the use case after merging results.

---

## ADR-006: FetchStrategy is resolved by the use case, not the repository

**Decision:** `GetWeatherSeriesUseCase` determines whether to call `/forecast`, `/archive`, or both. The repository executes a single endpoint call per invocation.

**Rationale:** The decision of which endpoint(s) to call is orchestration logic — it belongs in the use case. The repository's responsibility is to faithfully translate a query into an API call and map the response. Keeping the repository single-purpose makes it easier to test and reason about.

**Consequence:** For a date range that spans the 92-day boundary, the use case calls the repository twice and merges the resulting `WeatherReading` arrays, sorted by timestamp.

---

## ADR-007: IErrorReporter abstraction for error reporting

**Decision:** An `IErrorReporter` interface abstracts error reporting. `ConsoleErrorReporter` is the default implementation. A stub `SentryErrorReporter` is included (not wired) to demonstrate the pattern.

**Rationale:** Error reporting is a cross-cutting infrastructure concern that should be swappable without touching business logic. Encoding `console.error` or Sentry calls directly in use cases or composables would violate the dependency rule and make testing noisier.

**Current state:** `ConsoleErrorReporter` logs to console. `SentryErrorReporter` exists as a documented stub. The reporter is injected via `provide/inject` like other dependencies — replacing it in production requires one line change in `plugins/dependencies.ts`.

---

## ADR-008: Result type (option-t) for all async operations

**Decision:** All repository and use case methods return `Promise<Result<T, E>>`. Exceptions are never intentionally thrown across layer boundaries.

**Rationale:** Explicit error types make failure modes visible at the call site. TypeScript can enforce that the caller handles both `Ok` and `Err` cases. This is more robust than try/catch chains and more expressive than `null` returns.

**Rule:** If an unexpected exception escapes a repository (e.g. network error not caught by the HTTP client), the composable boundary catches it and converts it to `Err` before exposing it to components. Components never see raw exceptions.

---

## ADR-009: WeatherSeries embeds the originating WeatherQuery

**Decision:** `WeatherSeries.query` carries the full `WeatherQuery` that produced it.

**Rationale:** UI components need context to render labels: the unit symbol, the metric name, the aggregation disclaimer. Without embedding the query, this context would need to be passed separately through props or stored in additional reactive state. Embedding it keeps the data self-describing.
