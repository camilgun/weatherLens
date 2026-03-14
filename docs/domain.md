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
}
```

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

### WeatherReading

```typescript
type DataKind = 'historical' | 'forecast'

interface WeatherReading {
  readonly timestamp: Date
  readonly value: number        // always a single computed value — see Aggregation below
  readonly unit: WeatherUnit
  readonly kind: DataKind
  readonly aggregation: AggregationInterval
}
```

`DataKind` is a UI concern surfaced to the domain deliberately: the chart needs to visually distinguish historical from forecast data, and this is not an infrastructure detail.

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
type DailyAggregationMethod = 'mean_of_max_min' | 'sum' | 'mean' | 'max'

const dailyAggregationByMetric: Record<WeatherMetric, DailyAggregationMethod> = {
  temperature_2m:       'mean_of_max_min', // (max + min) / 2 — API does not provide mean
  precipitation:        'sum',             // precipitation_sum
  relative_humidity_2m: 'mean',            // relative_humidity_2m_mean (available in API)
  wind_speed_10m:       'max',             // wind_speed_10m_max
}
```

Used by:
- `GetWeatherSeriesUseCase` to compute `WeatherReading.value` for daily interval
- UI components to render the correct aggregation disclaimer (e.g. "Daily values represent mean of max/min temperatures")

**Important distinction:** `temperature_2m` daily is computed by the use case from two API values (`max` and `min`). `relative_humidity_2m` daily is a direct API value (`_mean` variable). The `DailyAggregationMethod` reflects the *semantic* meaning in both cases — the difference in origin is an infrastructure detail documented in IMPLEMENTATION_NOTES.md.

---

## Fetch Strategy

The domain defines the concept; the use case resolves it.

```typescript
type FetchStrategy = 'forecast_only' | 'archive_only' | 'both'
```

Resolution rules (applied by the use case, not the repository):

- `end <= today - 92 days` → `archive_only`
- `start >= today - 92 days` → `forecast_only`
- range spans the boundary → `both` (two API calls, results merged and sorted by timestamp)

The 92-day threshold is a named constant:

```typescript
const FORECAST_LOOKBACK_DAYS = 92
```

`DataKind` for each reading is determined independently of the fetch strategy:
- `timestamp < startOfToday` → `'historical'`
- `timestamp >= startOfToday` → `'forecast'`

This decoupling means the chart correctly labels data regardless of which endpoint it came from.

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
  fetchSeries(
    query: WeatherQuery,
    strategy: FetchStrategy
  ): Promise<Result<WeatherSeries, WeatherRepositoryError>>
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
- `dateRange.start < dateRange.end`
- `dateRange.end <= today + 16 days` (Open-Meteo forecast limit)
- `unit` is in `allowedUnitsByMetric[metric]`
- `location.latitude` and `location.longitude` are finite numbers in valid range

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
