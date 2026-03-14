# Domain

All types defined here are pure TypeScript — no runtime dependencies, no framework imports.

---

## Entities

### Location

```typescript
type LocationId = string // constructed as `lat:${lat},lon:${lon}`

interface Location {
  readonly id: LocationId
  readonly name: string
  readonly latitude: number
  readonly longitude: number
  readonly country?: string
  readonly timezone: string   // IANA timezone, e.g. "Europe/Rome"
}
```

`Location.timezone` is required because Open-Meteo timestamps must be interpreted in the selected location's timezone, not in the user's browser timezone. The use case also relies on it to distinguish historical vs forecast data correctly.

### WeatherMetric

```typescript
type WeatherMetric =
  | 'temperature_2m'
  | 'precipitation'
  | 'relative_humidity_2m'
  | 'wind_speed_10m'
```

### Units

```typescript
type TemperatureUnit    = 'celsius' | 'fahrenheit'
type PrecipitationUnit  = 'mm' | 'inch'
type WindUnit           = 'kmh' | 'ms' | 'mph' | 'kn'
type HumidityUnit       = 'percent'

type WeatherUnit = TemperatureUnit | PrecipitationUnit | WindUnit | HumidityUnit
```

Unit is modeled as a flat union at the type level. Constraint between metric and unit is enforced at runtime via `allowedUnitsByMetric` (see Validation Maps below).

Rationale: a generic `UnitForMetric<M>` is elegant but incompatible with Vue form state, which is always `string` at runtime. The validation map approach keeps the domain clean without fighting the framework.

### WeatherQuery

```typescript
type AggregationInterval = 'hourly' | 'daily'

interface WeatherQuery {
  readonly location: Location
  readonly dateRange: {
    readonly start: Date
    readonly end: Date
  }
  readonly metric: WeatherMetric
  readonly unit: WeatherUnit
  readonly interval: AggregationInterval
}
```

`WeatherQuery` is the single input to the weather use case. It must be fully validated before being passed downstream — see Validation below.

The date range is inclusive. `start` and `end` may refer to the same day, which represents a valid single-day query.

### RepositoryWeatherPoint

```typescript
interface RepositoryWeatherPoint {
  readonly timestamp: Date
  readonly unit: WeatherUnit
  readonly aggregation: AggregationInterval
  readonly value: number
}
```

`RepositoryWeatherPoint` is the repository-to-use-case boundary type. It is still a domain type, but it is not the final UI-ready reading model.

Purpose:
- keeps endpoint-specific raw field shapes and naming quirks inside infrastructure
- keeps `WeatherReading` clean and scalar for the chart and table
- gives the use case a normalized boundary model that it can merge, sort, and classify without knowing provider details

### WeatherReading

```typescript
type DataKind = 'historical' | 'forecast'

interface WeatherReading {
  readonly timestamp: Date
  readonly value: number        // final scalar value used by chart and table
  readonly unit: WeatherUnit
  readonly kind: DataKind
  readonly aggregation: AggregationInterval
}
```

`WeatherReading` is the final normalized domain model exposed by the use case. `DataKind` is a UI concern surfaced to the domain deliberately: the chart needs to visually distinguish historical from forecast data, and this is not an infrastructure detail.

### WeatherSeries

```typescript
interface WeatherSeries {
  readonly query: WeatherQuery   // carries context for labels and disclaimers
  readonly readings: ReadonlyArray<WeatherReading>
}
```

The query is embedded in the series so that UI components can derive labels (unit display, aggregation disclaimer) directly from the data, without needing separate state.

---

## Validation Maps

These maps live in `domain/entities/` and are the single source of truth for both validation logic and UI generation.

### allowedUnitsByMetric

```typescript
const allowedUnitsByMetric: Record<WeatherMetric, WeatherUnit[]> = {
  temperature_2m:       ['celsius', 'fahrenheit'],
  precipitation:        ['mm', 'inch'],
  relative_humidity_2m: ['percent'],
  wind_speed_10m:       ['kmh', 'ms', 'mph', 'kn'],
}
```

Used by:
- `validateWeatherQuery()` to reject invalid metric/unit combinations
- `WeatherConfig` component to dynamically render the unit selector options

### dailyAggregationByMetric

```typescript
type DailyAggregationMethod = 'sum' | 'mean' | 'max'

const dailyAggregationByMetric: Record<WeatherMetric, DailyAggregationMethod> = {
  temperature_2m:       'mean',            // use the provider's native daily mean when available
  precipitation:        'sum',             // precipitation_sum
  relative_humidity_2m: 'mean',            // relative_humidity_2m_mean (available in API)
  wind_speed_10m:       'max',             // wind_speed_10m_max
}
```

Used by:
- `GetWeatherSeriesUseCase` to keep the semantic meaning of daily values explicit when converting `RepositoryWeatherPoint` into final `WeatherReading`
- UI components to render the correct aggregation disclaimer (e.g. "Daily values represent the daily mean")

**Important distinction:** For the selected metrics in this challenge, daily values enter the domain as scalar values. `dailyAggregationByMetric` describes what that scalar means (`mean`, `sum`, `max`). If a specific Open-Meteo endpoint ever needs extra raw fields to obtain that scalar, that normalization stays in the infrastructure layer rather than leaking into the domain model.

---

## Fetch Strategy

The domain defines the concept; the use case resolves it.

```typescript
type FetchStrategy = 'forecast_only' | 'archive_only' | 'both'
type WeatherEndpoint = 'forecast' | 'archive'
```

Resolution rules (applied by the use case, not the repository):

- `end <= today - 92 days` → `archive_only`
- `start >= today - 92 days` → `forecast_only`
- range spans the boundary → `both` (two API calls, results merged and sorted by timestamp)

When the strategy is `both`, the use case must split the original query into two non-overlapping day-based subqueries because Open-Meteo filtering is date-based:

- the boundary day (`today - 92 days` in the selected location timezone) belongs entirely to `forecast`
- the `archive` subquery ends on the day **before** the boundary day
- the `forecast` subquery starts on the boundary day
- after clipping, any subquery with `start > end` is discarded instead of called

This avoids duplicate buckets and missing data around the 92-day limit.

The 92-day threshold is a named constant:

```typescript
const FORECAST_LOOKBACK_DAYS = 92
```

`DataKind` for each reading is determined independently of the fetch strategy, using the selected location's timezone:
- `hourly`: compare the reading instant against the current instant in the location timezone
- `daily`: compare the reading's local calendar day against today's local calendar day in the location timezone

This decoupling means the chart correctly labels data regardless of which endpoint it came from.

After fetching one or two endpoint results, the use case merges readings by timestamp, sorts ascending, and may deduplicate equal timestamps as a defensive safety net.

---

## Repository Interfaces

```typescript
// domain/repositories/IWeatherRepository.ts

interface WeatherRepositoryError {
  readonly kind: 'network' | 'parse' | 'invalid_query'
  readonly message: string
  readonly cause?: unknown
}

interface IWeatherRepository {
  fetchPoints(
    query: WeatherQuery,
    endpoint: WeatherEndpoint
  ): Promise<Result<ReadonlyArray<RepositoryWeatherPoint>, WeatherRepositoryError>>
}
```

```typescript
// domain/repositories/ILocationRepository.ts

interface LocationRepositoryError {
  readonly kind: 'not_found' | 'network'
  readonly message: string
}

interface ILocationRepository {
  search(query: string): Promise<Result<Location[], LocationRepositoryError>>
}
```

---

## Use Case Interfaces

```typescript
// domain/usecases/IGetWeatherSeriesUseCase.ts

interface GetWeatherSeriesError {
  readonly kind: 'repository' | 'invalid_query'
  readonly message: string
  readonly repositoryError?: WeatherRepositoryError
}

interface IGetWeatherSeriesUseCase {
  execute(query: WeatherQuery): Promise<Result<WeatherSeries, GetWeatherSeriesError>>
}
```

```typescript
// domain/usecases/ISearchLocationsUseCase.ts

interface ISearchLocationsUseCase {
  execute(query: string): Promise<Result<Location[], LocationRepositoryError>>
}
```

---

## Query Validation

```typescript
// domain/usecases/validateWeatherQuery.ts

interface WeatherQueryValidationError {
  readonly kind: 'invalid_date_range' | 'invalid_unit_for_metric' | 'invalid_location'
  readonly message: string
}

function validateWeatherQuery(
  query: WeatherQuery
): Result<WeatherQuery, WeatherQueryValidationError>
```

Validation rules:
- `dateRange.start <= dateRange.end`
- `dateRange.end <= today + 16 days` (Open-Meteo forecast limit)
- `unit` is in `allowedUnitsByMetric[metric]`
- `location.latitude` and `location.longitude` are finite numbers in valid range
- `location.timezone` is a non-empty IANA timezone string

This function is a pure domain function — no async, no side effects. Fully unit testable in isolation.

---

## Injection Keys

```typescript
// plugins/injectionKeys.ts

import type { InjectionKey } from 'vue'

const WEATHER_USECASE_KEY: InjectionKey<IGetWeatherSeriesUseCase> = Symbol('weatherUseCase')
const LOCATION_USECASE_KEY: InjectionKey<ISearchLocationsUseCase> = Symbol('locationUseCase')
const ERROR_REPORTER_KEY: InjectionKey<IErrorReporter> = Symbol('errorReporter')
```

Typed `InjectionKey` ensures `inject()` calls return the correct type without casting.
