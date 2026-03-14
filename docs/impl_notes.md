# Implementation Notes

Operational details, non-obvious decisions, and things that must be explicitly commented in the code. Organized by layer.

---

## infrastructure/api/openmeteo/

### Endpoint structure

Two base URLs:
- `https://api.open-meteo.com/v1/forecast` — covers past 92 days + up to 16 days ahead
- `https://archive-api.open-meteo.com/v1/archive` — historical data beyond 92 days

Both accept the same core parameters but differ in available variables (see below).

### API parameter naming — must match exactly

These are the canonical Open-Meteo parameter names for the metrics chosen in this challenge. They differ from the domain entity names and are easy to get wrong:

| Domain concept | API parameter (hourly) | API parameter (daily) |
|---|---|---|
| `temperature_2m` | `temperature_2m` | `temperature_2m_mean` |
| `precipitation` | `precipitation` | `precipitation_sum` |
| `relative_humidity_2m` | `relative_humidity_2m` | `relative_humidity_2m_mean` |
| `wind_speed_10m` | `wind_speed_10m` | `wind_speed_10m_max` |

Unit parameter names also differ from each other:
- `temperature_unit`: `celsius` or `fahrenheit`
- `precipitation_unit`: `mm` or `inch`
- `wind_speed_unit`: `kmh`, `ms`, `mph`, `kn`
- Humidity has no unit parameter (always percent)

**Comment to add in the mapper:** The mapper normalizes provider-specific field shapes into a scalar `RepositoryWeatherPoint.value`. It should not know about `DataKind`, fetch strategy, or current-time comparisons. If one endpoint ever needs multiple raw fields to produce the scalar, that normalization still belongs here, not in the domain model.

### Timezone handling

When `interval === 'daily'`, the API requires a `timezone` parameter. In this project, the repository should send `timezone` for **both** `daily` and `hourly` requests using `query.location.timezone`.

Rules:
- `Location.timezone` comes from the Geocoding API and is stored in the domain model
- Weather requests pass `timezone: query.location.timezone`
- Do not rely on the browser timezone
- Avoid `timezone: 'auto'` in application code; an explicit timezone from the selected location is more deterministic and easier to test

**Comment to add in the HTTP client:** `timezone` is always sent from `query.location.timezone`. Daily requests require it, and hourly requests also use it so returned timestamps stay aligned with the selected location instead of the browser environment.

### metricApiSpec mapper

This is the infrastructure function that translates `{ endpoint, metric, interval }` → `{ apiVariables: string[], requiresTimezone: boolean }`.

```
endpoint: 'forecast', metric: 'temperature_2m', interval: 'hourly'
  → { variables: ['temperature_2m'], requiresTimezone: false }

endpoint: 'forecast', metric: 'temperature_2m', interval: 'daily'
  → { variables: ['temperature_2m_mean'], requiresTimezone: true }

endpoint: 'forecast', metric: 'relative_humidity_2m', interval: 'daily'
  → { variables: ['relative_humidity_2m_mean'], requiresTimezone: true }
```

**Comment to add:** This function is the only place in the codebase that knows about Open-Meteo's variable naming conventions. If different endpoints ever require different variable selections for the same domain metric, keep that asymmetry here instead of leaking it into the domain layer.

### Raw response shape

The API returns timestamps as ISO strings in the `time` array, values in a parallel array keyed by variable name:

```json
{
  "hourly": {
    "time": ["2024-01-01T00:00", "2024-01-01T01:00"],
    "temperature_2m": [4.2, 3.8]
  }
}
```

For daily:
```json
{
  "daily": {
    "time": ["2024-01-01", "2024-01-02"],
    "temperature_2m_mean": [4.6, 3.05]
  }
}
```

Important: these timestamps are expressed in the timezone requested from Open-Meteo. They must be parsed with the selected location's timezone, not with `new Date(rawString)` in the browser timezone.

The mapper zips `time[]` with value array(s) into `RepositoryWeatherPoint[]`.

- Hourly metrics always produce one scalar `value`
- Daily metrics for this challenge also produce one scalar `value` per timestamp
- If an endpoint ever needs multiple raw fields to derive that scalar, the normalization stays in infrastructure before creating the repository point

**Comment to add in mapper:** The mapper applies provider-shape normalization only, including timezone-aware timestamp parsing. Domain decisions such as `DataKind` assignment stay in the use case.

---

## domain/usecases/

### GetWeatherSeriesUseCase

**FetchStrategy resolution:**

```typescript
const boundaryDate = subDays(startOfToday(), FORECAST_LOOKBACK_DAYS) // 92 days

if (query.dateRange.end <= boundaryDate)   → 'archive_only'
if (query.dateRange.start >= boundaryDate) → 'forecast_only'
else                                       → 'both'
```

**Comment to add:** `FORECAST_LOOKBACK_DAYS = 92` is a named constant because Open-Meteo's lookback window could change. It must not be a magic number.

After resolving `FetchStrategy`, the use case maps it to one or two repository calls:

```typescript
if (strategy === 'archive_only')  call fetchPoints(query, 'archive')
if (strategy === 'forecast_only') call fetchPoints(query, 'forecast')
if (strategy === 'both')          call both and merge
```

**Comment to add:** The repository does not know about `FetchStrategy`. It accepts exactly one endpoint per invocation. The use case owns the orchestration.

### Range splitting at the 92-day boundary

Open-Meteo requests are day-based (`start_date`, `end_date`), so when a query spans the 92-day limit the split must also be day-based.

Rule to implement:
- the boundary day belongs entirely to `forecast`
- `archive` ends on `boundaryDate - 1 day`
- `forecast` starts on `boundaryDate`
- empty subqueries after clipping are skipped

Suggested helper:

```typescript
function splitQueryByEndpoint(query: WeatherQuery, boundaryDate: Date) {
  const archiveQuery =
    query.dateRange.start <= subDays(boundaryDate, 1)
      ? {
          ...query,
          dateRange: {
            start: query.dateRange.start,
            end: minDate(query.dateRange.end, subDays(boundaryDate, 1)),
          },
        }
      : null

  const forecastQuery =
    query.dateRange.end >= boundaryDate
      ? {
          ...query,
          dateRange: {
            start: maxDate(query.dateRange.start, boundaryDate),
            end: query.dateRange.end,
          },
        }
      : null

  return {
    archiveQuery,
    forecastQuery,
  }
}
```

**Comment to add:** The boundary day is intentionally owned by `forecast`. This prevents overlap because the API range parameters are day-based, not timestamp-based.

**Daily value semantics:**

After receiving normalized readings from the repository, the use case carries the scalar values into final `WeatherReading` objects:

- `mean`: daily temperature and humidity values represent a daily mean
- `sum`: daily precipitation values represent the accumulated daily total
- `max`: daily wind values represent the daily maximum

**Comment to add:** `dailyAggregationByMetric` encodes the semantic meaning of the scalar daily value. It drives reviewer-facing clarity and UI disclaimers, while provider-specific raw-field normalization stays in infrastructure.

**DataKind assignment:**

After merging all readings (from one or two API calls), assign `kind` using the selected location's timezone:

```typescript
if (query.interval === 'hourly') {
  reading.kind = reading.timestamp < nowInLocation ? 'historical' : 'forecast'
}

if (query.interval === 'daily') {
  reading.kind = readingLocalDay < todayLocalDay ? 'historical' : 'forecast'
}
```

**Comment to add:** `DataKind` is assigned here, after merging, so it reflects the user's temporal reality in the selected location — not the API endpoint that served the data and not the developer's browser timezone. A reading from `/forecast` with a past timestamp is still `'historical'`.

---

## composables/

### Inject pattern

Every composable that needs a use case should use a typed wrapper around `inject()`:

```typescript
function injectOrThrow<T>(key: InjectionKey<T>, name: string): T {
  const value = inject(key)
  if (value === undefined) throw new Error(`[DI] ${name} not provided. Did you register the plugin?`)
  return value
}
```

**Comment to add on this function:** This is a development-time safety net, not error handling. If this throws, it means `plugins/dependencies.ts` was not loaded — it should never throw in production.

### TanStack Query key structure

Use a consistent, typed key factory to avoid cache collisions:

```typescript
const weatherKeys = {
  series: (query: WeatherQuery) => ['weather', 'series', query] as const,
}
const locationKeys = {
  search: (q: string) => ['location', 'search', q] as const,
}
```

**Comment to add:** Keys include the full `WeatherQuery` object. TanStack Query serializes it for comparison. This means changing any field (metric, date, unit) correctly invalidates the cache.

### Result unwrapping

The composable is responsible for converting `Result<T, E>` into Vue-reactive state:

```typescript
// pattern to follow in every useCase composable:
const { data, ...rest } = useQuery({
  queryKey: ...,
  queryFn: async () => {
    const result = await useCase.execute(query)
    if (isErr(result)) {
      errorReporter.report(unwrapErr(result))
      throw unwrapErr(result) // TanStack Query needs a throw to set error state
    }
    return unwrapOk(result)
  }
})
```

**Comment to add:** The `throw` here is intentional. TanStack Query's error state is set by thrown exceptions in `queryFn`. We convert the `Err` result to a throw at the composable boundary — this is the only place in the app where we intentionally throw.

---

## lib/errorReporting/

### SentryErrorReporter stub

```typescript
// SentryErrorReporter.ts
// NOT wired in production — kept as a documented integration point.
// To enable: npm install @sentry/vue, initialize Sentry in main.ts,
// then replace ConsoleErrorReporter with SentryErrorReporter in plugins/dependencies.ts.
export class SentryErrorReporter implements IErrorReporter {
  report(error: unknown, context?: Record<string, unknown>): void {
    // Sentry.captureException(error, { extra: context })
    throw new Error('SentryErrorReporter is not yet configured')
  }
}
```

---

## components/

### Aggregation disclaimer

The `WeatherDashboard` or `WeatherChart` component must display a disclaimer when `query.interval === 'daily'`. The disclaimer text is derived from `dailyAggregationByMetric[query.metric]`:

```typescript
const disclaimerByMethod: Record<DailyAggregationMethod, string> = {
  sum:             'Daily values show the total accumulated amount',
  mean:            'Daily values show the daily mean',
  max:             'Daily values show the daily maximum',
}
```

This map lives in the component layer (presentation concern), but it derives from the domain map. If a new metric is added to `dailyAggregationByMetric`, TypeScript will surface a missing key here.

### Chart: visual distinction historical vs forecast

The chart splits `WeatherSeries.readings` into two datasets by `kind`:

```typescript
const historical = readings.filter(r => r.kind === 'historical')
const forecast   = readings.filter(r => r.kind === 'forecast')
```

Render as two series on the same axis (e.g. solid line for historical, dashed for forecast, different color). The split is done in the component, not in the domain or composable — it is purely a rendering decision.

---

## Unit Tests — What to Cover

| Target | Type | Priority |
|---|---|---|
| `validateWeatherQuery()` | Pure function | High — covers all invalid combinations |
| `openmeteo/mapper` | Pure function | High — snapshot of raw response → RepositoryWeatherPoint[] |
| `metricApiSpec` mapper | Pure function | High — verifies correct API variable names |
| `GetWeatherSeriesUseCase` | Unit with mock repo | High — FetchStrategy, boundary split, aggregation, DataKind assignment |
| `dailyAggregationByMetric` semantics | Pure logic | Medium |
| `injectOrThrow` | Pure function | Low — simple guard |
| Composables | Integration with Vue Test Utils | Low — covered implicitly by use case tests |
| Components | Mount tests | Out of scope for this challenge |

**Comment to add in each test file:** Each test file should have a one-line comment at the top stating what layer/contract it tests and why it can be tested in isolation. This makes the test structure legible to a reviewer scanning the repo.

Boundary-specific tests worth keeping:
- query entirely before the boundary → one `archive` call
- query entirely from the boundary day onward → one `forecast` call
- query spanning the boundary → two calls, clipped with no overlap
- query ending exactly on `boundaryDate - 1 day` → archive only
- query starting exactly on `boundaryDate` → forecast only
