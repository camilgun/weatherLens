# Architecture

## Overview

Single-page application built with Vue 3. The codebase is organized in concentric layers: **domain** at the center (no dependencies), **infrastructure** and **application** around it, **composables** as the boundary into Vue, and **components** at the outermost edge.

Data flows in one direction:

```
Open-Meteo API
  → infrastructure/api (raw response)
  → infrastructure/repositories (mapped to domain entities)
  → domain/usecases (business logic applied)
  → composables (TanStack Query + Vue reactivity)
  → components (pure presentation)
```

No raw API response ever crosses the infrastructure boundary. Components only consume domain entities.

---

## Folder Structure

```
src/
├── domain/
│   ├── entities/               # Pure domain types — no imports from outside domain/
│   ├── repositories/           # Repository interfaces (contracts)
│   └── usecases/               # Use case interfaces + concrete implementations
│
├── infrastructure/
│   ├── api/
│   │   ├── openmeteo/          # HTTP client + raw response types + mappers
│   │   └── geocoding/          # Open-Meteo Geocoding API client + mappers
│   └── repositories/           # Concrete implementations of domain repository interfaces
│
├── composables/                # Vue boundary: inject use cases + wrap with TanStack Query
│
├── components/
│   ├── ui/                     # Presentational, stateless components (chart, table, etc.)
│   └── features/               # Smart components — consume composables, own layout
│       ├── LocationSelector/
│       ├── WeatherConfig/
│       └── WeatherDashboard/
│
├── lib/
│   ├── errorReporting/         # IErrorReporter abstraction + ConsoleErrorReporter impl
│   └── result/                 # Re-exports and helpers for option-t Result type
│
├── plugins/
│   └── dependencies.ts         # Factory functions + App.provide() — DI composition root
│
└── main.ts
```

---

## Layer Rules

### `domain/`
- Zero external dependencies (no Vue, no axios, no option-t internals beyond types)
- No `async` side effects — pure functions and type definitions only
- Everything else depends on this layer; this layer depends on nothing

### `infrastructure/`
- Depends on `domain/` (implements its interfaces, produces its entities)
- Knows about HTTP, JSON shapes, API quirks
- Never imported by `components/` or `composables/` directly — only via interfaces

### `domain/usecases/`
- Interfaces live here alongside implementations (pragmatic collapsing of domain/application layers — see DECISIONS.md)
- Depend on repository interfaces, never on concrete implementations
- Apply business logic: daily aggregation calculations, fetch strategy resolution, data merging

### `composables/`
- The only layer allowed to use Vue APIs (`inject`, `ref`, `computed`) and TanStack Query
- Inject use cases via `inject()` — never instantiate them directly
- Return typed Vue-reactive data to components; hide all async/error complexity

### `components/`
- Consume composables or receive props — no direct use case or repository access
- `ui/` components are fully stateless and reusable
- `features/` components own layout and wire composables to `ui/` components

### `plugins/dependencies.ts`
- The single place where concrete classes are instantiated
- Called once in `main.ts` via `app.use(dependenciesPlugin)`
- All `app.provide()` calls live here — no `provide()` anywhere else in the app

---

## Data Flow: Weather Series Request

```
1. User fills WeatherConfig form
   → WeatherQuery domain entity assembled + validated

2. useWeatherSeries composable (TanStack Query)
   → calls IGetWeatherSeriesUseCase.execute(query)

3. GetWeatherSeriesUseCase
   → determines FetchStrategy from dateRange (see DOMAIN.md)
   → calls IWeatherRepository.fetchSeries() once or twice
   → merges results if strategy is 'both'
   → applies daily aggregation logic if interval === 'daily'
   → returns Result<WeatherSeries, WeatherRepositoryError>

4. IWeatherRepository (OpenMeteoWeatherRepository)
   → builds API params from query (metric + interval → API variable names)
   → calls /forecast and/or /archive endpoint
   → maps raw response to WeatherReading[]
   → returns Result<WeatherSeries, WeatherRepositoryError>

5. Composable unwraps Result
   → exposes { data, isLoading, error } to components

6. WeatherDashboard
   → passes WeatherSeries to WeatherChart (ui/) and WeatherTable (ui/)
   → chart uses reading.kind to visually distinguish historical vs forecast
```

---

## Dependency Injection Strategy

Factory functions create all dependencies. `App.provide()` registers them under typed injection keys.

```
dependenciesPlugin
  → creates OpenMeteoApiClient
  → creates OpenMeteoWeatherRepository(apiClient)
  → creates OpenMeteoLocationRepository(apiClient)
  → creates GetWeatherSeriesUseCase(weatherRepository)
  → creates SearchLocationsUseCase(locationRepository)
  → creates ConsoleErrorReporter
  → app.provide(WEATHER_USECASE_KEY, getWeatherSeriesUseCase)
  → app.provide(LOCATION_USECASE_KEY, searchLocationsUseCase)
  → app.provide(ERROR_REPORTER_KEY, errorReporter)
```

Composables call `inject(WEATHER_USECASE_KEY)` — they never import concrete classes.

This makes every use case swappable in tests by calling `app.provide()` with a mock before mounting.

---

## Error Handling Flow

All async operations return `Result<OkValue, ErrorValue>` (option-t).

```
Repository         → returns Result<WeatherSeries, WeatherRepositoryError>
UseCase            → propagates or transforms Result
Composable         → unwraps Result:
                       Ok  → sets TanStack Query data
                       Err → calls IErrorReporter.report() + exposes typed error to component
Component          → renders error state from typed error, never catches exceptions
```

Exceptions that escape (unexpected runtime errors) are caught at the composable boundary and converted to `Err` before reaching components.
