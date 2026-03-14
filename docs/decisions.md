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

**Trade-off:** The type system cannot catch an invalid metric/unit combination at compile time. This is accepted because `validateWeatherQuery(query, now)` catches it at runtime before the query reaches the use case, and the form dynamically renders only valid unit options from `allowedUnitsByMetric`.

---

## ADR-004: Repository returns an intermediate weather-point model; use case produces the final reading model

**Decision:** `WeatherReading.value` is always a single number in the final domain model, but the repository does not return `WeatherReading` directly. It returns `RepositoryWeatherPoint`, an intermediate domain type that is already normalized to a scalar `value`. Any endpoint-specific raw field asymmetry is resolved in infrastructure before the data crosses into the domain, and `null` upstream values are rejected before a repository point is created.

**Rationale:** The chart and table both need a scalar value per timestamp. At the same time, Open-Meteo endpoint details such as field naming or per-endpoint variable differences should not leak into the use case. `RepositoryWeatherPoint` keeps that normalization concern inside infrastructure while still preventing the repository from owning domain concerns like `DataKind`.

**Alternative considered:** Return `WeatherReading` directly from the repository. Rejected because that would force the repository to know about current-time semantics and assign `historical` vs `forecast`, which belongs in the use case.

**Trade-off:** The repository/use-case boundary becomes one step more explicit. This adds a small amount of type surface area, but it keeps provider normalization localized and leaves the use case focused on orchestration: fetch strategy resolution, merging, sorting, and `DataKind` assignment.

---

## ADR-005: DataKind is a domain concept, not an infrastructure detail

**Decision:** `WeatherReading.kind: DataKind` ('historical' | 'forecast') is determined from the selected location's timezone, not from the browser timezone and not from which API endpoint produced the reading. The current instant/day used for this comparison comes from an injected time source, not from ambient global time.

**Rationale:** The visual distinction between historical and forecast data is meaningful to the user and belongs in the domain model. Coupling it to which endpoint was called would leak infrastructure structure into the UI, and using the browser timezone would mislabel data for locations outside the user's local timezone. Injecting the time source also makes these rules deterministic in tests.

**Rule:**
- `hourly`: compare the reading instant against the current instant in the selected location's timezone
- `daily`: compare the reading's local calendar day against today's local calendar day in the selected location's timezone

This means an hourly reading from earlier today is still `'historical'`, while a daily bucket for today is treated as `'forecast'`.

---

## ADR-006: FetchStrategy is resolved by the use case, not the repository

**Decision:** `GetWeatherSeriesUseCase` determines whether the query needs `forecast`, `archive`, or both. `IWeatherRepository` does not accept `FetchStrategy`; it accepts a single `WeatherEndpoint` (`'forecast' | 'archive'`) and executes exactly one endpoint call per invocation.

**Rationale:** The decision of which endpoint(s) to call is orchestration logic — it belongs in the use case. The repository's responsibility is to faithfully translate a query into an API call and map the response. Keeping the repository single-purpose makes it easier to test and reason about.

**Boundary rule:** The 92-day boundary is computed from the current local calendar day in the selected location timezone, using an injected time source. `WeatherQuery.dateRange` is modeled as `LocalDate` (`YYYY-MM-DD`), not `Date`, so the split operates on location-local calendar days rather than browser-timezone instants. If the range spans that limit, the boundary day belongs entirely to the `forecast` endpoint and the `archive` subquery ends on the previous day. This keeps the split day-based, which matches Open-Meteo's `start_date` / `end_date` semantics, and avoids overlap.

**Consequence:** `FetchStrategy` remains a use-case concern only. `archive_only` applies when `end < boundaryDay`; `forecast_only` applies when `start >= boundaryDay`; otherwise the use case performs two repository calls, one with `'archive'` and one with `'forecast'`, clips the subqueries so they do not overlap, then merges the resulting `RepositoryWeatherPoint` arrays, sorted by timestamp, before converting them into final `WeatherReading` values. A timestamp-level dedupe after merge is acceptable as a defensive safeguard.

---

## ADR-007: IErrorReporter abstraction for error reporting

**Decision:** An `IErrorReporter` interface abstracts error reporting. `ConsoleErrorReporter` is the default implementation. A `SentryErrorReporter` adapter template is included (not wired) to demonstrate the pattern, but it degrades safely until Sentry is configured.

**Rationale:** Error reporting is a cross-cutting infrastructure concern that should be swappable without touching business logic. Encoding `console.error` or Sentry calls directly in use cases or composables would violate the dependency rule and make testing noisier.

**Current state:** `ConsoleErrorReporter` logs to console. `SentryErrorReporter` exists as a documented integration template with a `console.warn` fallback so it cannot mask the original application error if it is wired accidentally before Sentry setup is complete. The reporter is injected via `provide/inject` like other dependencies — replacing it in production requires one line change in `plugins/dependencies.ts`.

---

## ADR-008: Result type (option-t) for all async operations

**Decision:** All repository and use case methods return `Promise<Result<T, E>>`. Exceptions are never intentionally thrown across layer boundaries.

**Rationale:** Explicit error types make failure modes visible at the call site. TypeScript can enforce that the caller handles both `Ok` and `Err` cases. This is more robust than try/catch chains and more expressive than `null` returns. The `Result` type boundary is introduced together with the domain contracts via a small local facade (`lib/result`) so the interfaces are complete from Task 2 onward, while helper functions can be added later where they are operationally needed.

**Rule:** If an unexpected exception escapes a repository (e.g. network error not caught by the HTTP client), the composable boundary catches it and converts it to `Err` before exposing it to components. Components never see raw exceptions.

---

## ADR-009: WeatherSeries embeds the originating WeatherQuery

**Decision:** `WeatherSeries.query` carries the full `WeatherQuery` that produced it.

**Rationale:** UI components need context to render labels: the unit symbol, the metric name, the aggregation disclaimer. Without embedding the query, this context would need to be passed separately through props or stored in additional reactive state. Embedding it keeps the data self-describing.

---

## ADR-010: Inject a time source for all time-dependent rules

**Decision:** Time-dependent use cases receive an `IClock` abstraction. Pure validation helpers receive `now: Date` explicitly as a parameter.

**Rationale:** The project has multiple rules that depend on "today" or "now": the 92-day forecast/archive split, the 16-day forecast limit, and `DataKind` assignment. Reading ambient time directly from the runtime would couple those rules to the browser or CI environment and make timezone-sensitive tests brittle.

**Trade-off:** This adds a small amount of plumbing in the composition root and constructor signatures. The benefit is that time semantics become explicit, deterministic, and reviewable.
