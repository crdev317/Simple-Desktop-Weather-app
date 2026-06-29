# Feature 1 — Show Current Conditions for one location (tracer bullet)

**Context references:**
- `business-domain-context.md` (the project's domain glossary)
- `Technical-Context.MD`
- `PRD.md`
- `Roadmap.md` → Feature 1: Show Current Conditions for one location
- `docs/adr/` — empty at time of writing (no ADRs yet)

> Authority order (lower wins): ADR > Technical-Context > business-domain-context > PRD > Roadmap > Spec > Plan.

## Purpose

The tracer bullet. On launch, the app fetches and displays the **Current Conditions**
(the present **Reading**) for a single hard-coded **Location** from Open-Meteo, showing
temperature and a readable **Condition** in a WPF window. Its job is to pierce every layer
end-to-end — WPF view ↔ view-model ↔ Weather Provider (Open-Meteo HTTP) ↔ JSON→Reading ↔
Condition Mapper ↔ Serilog — and to stand up the three-tier testing harness, with nothing
else in the way.

## Scope

**In scope**
- Fetch Current Conditions for one hard-coded Location (London, `51.51, -0.13`) on launch.
- Display: hard-coded location label · temperature in °C · Condition as readable text · "as of HH:mm" (the Reading's observation time).
- A terminal, plain-language **"Couldn't load the weather."** state on any fetch failure — the app must never crash (Overriding Principle #3).
- Async, cancellable network I/O off the UI thread (Overriding Principle #4).
- Serilog structured logging of every request and every failure to a rolling file.

**Out of scope** (deferred to later Features, per the Roadmap)
- Place Search, the Saved Locations list, Default/Active switching (Features 2–3).
- Forecast (Feature 4).
- Unit Preference — **fixed to metric (°C)** for this Feature (Feature 5).
- The full **Stale**/**Last-Known Reading** fallback and any retry/refresh — a basic
  "couldn't load" message is the whole failure UX here (Feature 6). A failure is terminal
  until app restart (no Retry button — explicit decision).
- Condition **icons** — readable text only in v1 (the PRD's open question, resolved to text).

## Decisions taken during brainstorming

1. **Fetch-once, no Retry.** The view-model exposes one async load command fired on launch.
   A failed launch fetch leaves a terminal "couldn't load" message; Retry/refresh is Feature 6.
2. **Hard-coded Location = London `51.51, -0.13`**, carried in `appsettings.json` bound to a
   `LocationOptions` record — a single, obvious seam for Feature 2's Place Search to replace,
   not a literal buried in the view-model.
3. **Display surface** = label · °C · Condition text · "as of HH:mm". Timestamp included
   because it is nearly free and seeds the Stale/Last-Known display Feature 6 builds on.
4. **Solution shape = three projects** (see Architecture) so the "view-models have no
   UI-thread dependency" rule in `Technical-Context.MD` is enforced by the compiler, not convention.

## Architecture

Three projects in one solution:

```
WeatherApp.Core      (net10.0, no WPF)   — domain + provider + mapper + view-model
WeatherApp           (net10.0-windows)   — Views + App bootstrap/DI, references Core
WeatherApp.Tests     (net10.0)           — references Core only
```

`App.xaml.cs` builds a generic `Host` (`Microsoft.Extensions.Hosting`), registers services in
DI (`AddHttpClient` for the typed Open-Meteo client with a base address + bounded timeout, the
`ConditionMapper`, the `CurrentConditionsViewModel`, `LocationOptions` bound from config, and
Serilog), resolves `MainWindow`, sets its `DataContext` to the view-model, and shows it.

The `WeatherApp.Tests` project cannot reference `System.Windows`, so any view-model that reached
for a UI-thread type would fail to compile there — turning Overriding Principle's testability
requirement into a build-enforced boundary.

## Components

| Unit | Project | Responsibility | Depends on |
|---|---|---|---|
| `Coordinates` (record) | Core | lat/lon value object — Location identity per the glossary. | — |
| `Reading` (record) | Core | Immutable observed snapshot: temperature (°C, `double`), WMO code (`int`), observation time (`DateTimeOffset`/local string). Pure data; never a prediction. | — |
| `Condition` (record) | Core | Display Condition: readable text + a stable icon key (icon unused in v1, reserved). | — |
| `IWeatherProvider` | Core | `Task<Reading> GetCurrentConditionsAsync(Coordinates coords, CancellationToken ct)`. | — |
| `OpenMeteoWeatherProvider` | Core | The one implementation: builds the Open-Meteo URL, `GET`s it (logged), deserializes JSON into a private DTO, maps WMO code via `ConditionMapper`, returns a `Reading`. Async + cancellable. | `HttpClient`, `ConditionMapper`, `ILogger` |
| `ConditionMapper` | Core | Pure: WMO weather code → `Condition`. Total over the WMO code set with an explicit "unknown code" fallback (never throws). | — |
| `CurrentConditionsViewModel` | Core | `CommunityToolkit.Mvvm` `ObservableObject`. Exposes display state (label, temperature string, condition text, "as of HH:mm") and a `Status` (`Loading`/`Loaded`/`Failed`). One `LoadAsync` async-relay-command, fired on launch. No UI-thread types. | `IWeatherProvider`, `LocationOptions` |
| `LocationOptions` (record) | Core | `{ double Latitude; double Longitude; string Label; }` — the hard-coded Location, bound from config. | — |
| `MainWindow` (+ XAML) | WPF | Binds the view-model; renders the four fields when `Loaded`, "Loading…" when `Loading`, "Couldn't load the weather." when `Failed`. Triggers `LoadAsync` on window load. | view-model |
| Serilog bootstrap | WPF | Rolling file sink under `%LOCALAPPDATA%\SimpleDesktopWeatherApp\logs\`. | — |

## Data flow (launch → screen)

1. `App` starts the Host, builds DI, resolves and shows `MainWindow`.
2. Window load triggers `CurrentConditionsViewModel.LoadAsync(ct)` (status → `Loading`).
3. View-model calls `IWeatherProvider.GetCurrentConditionsAsync(londonCoords, ct)`.
4. Provider builds the URL, `GET`s it (INFO log: URL + status + latency), deserializes JSON,
   maps the WMO code via `ConditionMapper`, returns a `Reading`.
5. View-model maps the `Reading` into display strings (°C, condition text, "as of HH:mm"),
   status → `Loaded`.
6. The View binds and renders. The fetch is awaited off the UI thread throughout, so the
   window never blocks (Overriding Principle #4).

## Error handling (Overriding Principle #3 — the UI never crashes on a failed/slow fetch)

`LoadAsync` is the single choke point. It wraps the provider call in try/catch and translates
**every** failure into observable state — never an exception that reaches the UI:

- **Status model:** `Loading` → `Loaded` | `Failed`. The window shows "Loading…" while
  `Loading`, the four fields while `Loaded`, and **"Couldn't load the weather."** while `Failed`.
- **What's caught:** `HttpRequestException` (network / DNS / non-success status),
  `TaskCanceledException` / timeout, and `JsonException` / malformed-or-missing-field parse
  failures — all collapse to the single `Failed` state. The typed `HttpClient` sets a bounded
  `Timeout` so a hung connection becomes `Failed`, not a frozen window.
- **What is never shown:** error codes, HTTP statuses, stack traces, raw payload (User Feedback
  section of `Technical-Context.MD`). Those go to the log only.
- **What's logged:** every request as INFO (URL + status + latency); every failure as ERROR
  with the exception + stack trace.

This contract — the provider may throw, the view-model catches everything and yields `Failed` —
is an in-process boundary (not one of the seven seam channel classes), so it carries no
`## Seam inventory` entry; it is pinned directly by the view-model Tier-1 test below.

## Testing (three-tier standard; "test the contract, not the provider")

| Unit | Tier | Asserted |
|---|---|---|
| `OpenMeteoWeatherProvider` | **Tier 1 — recorded-replay** (`WireMock.Net` serving the captured fixture over a real socket) | The captured payload parses into the expected `Reading` shape (temp `double`, WMO `int`, time); a non-success status and a malformed/short body each surface as the provider's documented failure (exception caught upstream). Real HTTP on one side. |
| `OpenMeteoWeatherProvider` | **Tier 2 — live, disposable** | One real call to Open-Meteo for London returns a parseable `Reading`. Single request, scheduled, keyless → Tier-2 cost ceiling ≈ nil. Proves the live seam the fixture cannot. |
| `ConditionMapper` | **Tier 1** | Table-driven over the WMO code set: known codes → expected text; an unmapped code → the explicit fallback, no throw. |
| `CurrentConditionsViewModel` | **Tier 1** | Off the UI thread with a fake `IWeatherProvider`: success → `Loaded` + correct display strings; thrown exception / cancellation → `Failed`, no exception escapes. Asserts observable state transitions, not internals. |
| `LocationOptions` binding | **Tier 1** | A sample `appsettings.json` section binds to the expected `LocationOptions`; a missing/empty section fails fast with a clear error rather than silent `(0,0,null)`. |

**Feature-level test plan:** Tier 1 runs every commit; Tier 2 is scheduled (off the PR gate)
with the one-call London ceiling; Tier 3 (packaged-app smoke) is manual for this Feature.
**Fixtures needed:** one captured Open-Meteo current-weather JSON response for London — captured
live during brainstorming (verbatim in Seam 1 below); the Plan/TDD step writes it into the test
project as the Tier-1 replay fixture.

## Seam inventory

### Seam 1: Open-Meteo current-weather HTTP
- **(a) class:** `network-protocol`, **external**. (Data-format facet rides on this channel — see (c).)
- **(b) sides:** `OpenMeteoWeatherProvider` (our code) ↔ Open-Meteo Forecast API (`api.open-meteo.com`).
- **(c) contract:**
  - **Request:** `GET https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current=temperature_2m,weather_code&timezone=GMT`.
  - **Auth (first contact — pinned):** Open-Meteo offers two tiers: a **keyless** free tier for
    non-commercial use on `api.open-meteo.com` (no API key, no header, no token), and a commercial
    tier on `customer-api.open-meteo.com` using an `&apikey=` query parameter. **We use the keyless
    free tier** — chosen because the app is non-commercial and low-volume; no credential is stored,
    so Overriding Principle #1 (Windows Credential Manager) is not engaged until/unless a keyed
    tier is adopted (which would require an ADR). Later Open-Meteo seams (geocoding in Feature 2,
    forecast in Feature 4) inherit this keyless decision by reference.
  - **Success shape (HTTP 200, `application/json`):** a top-level object with a non-null
    `current` object containing `temperature_2m` (non-null `number`, °C), `weather_code` (non-null
    integer, WMO code), and `time` (non-null ISO-8601 `string`, no seconds, in the requested
    timezone). Field names are **snake_case**, so `System.Text.Json` needs explicit
    `JsonPropertyName` mappings (or snake_case policy). Top-level `latitude`/`longitude` are the
    **grid-snapped** coordinates and **may differ** from those requested — the app ignores them and
    displays its own hard-coded label, so snapping does not affect display.
  - **Failure shape:** on bad parameters the API returns **HTTP 400** with
    `{"error":true,"reason":<string>}`. Any non-2xx, a timeout, or a body missing `current` /
    its fields is treated by the provider as a failed fetch (surfaces to the view-model's `Failed`).
  - **Nullability:** when `current=temperature_2m,weather_code` is requested and the call
    succeeds, `current` and the three fields are present; the provider must still treat their
    absence as failure (not assume presence).
- **(d) proof:** Tier-1 `WireMock.Net` replay of the captured fixture asserting the parse into
  `Reading` (real HTTP on the WireMock side) + a malformed-body case; Tier-2 live single call.
  **Captured live payload (London, 2026-06-29, HTTP 200) — the Tier-1 fixture:**
  ```json
  {"latitude":51.5,"longitude":-0.25,"generationtime_ms":0.35,"utc_offset_seconds":0,
   "timezone":"GMT","timezone_abbreviation":"GMT","elevation":29.0,
   "current_units":{"time":"iso8601","interval":"seconds","temperature_2m":"°C","weather_code":"wmo code"},
   "current":{"time":"2026-06-29T14:15","interval":900,"temperature_2m":23.6,"weather_code":3}}
  ```
- **(e) authority:** Supervised read-only live call to `api.open-meteo.com/v1/forecast`,
  observed 2026-06-29 (payload above), cross-checked against the Open-Meteo Forecast API docs
  (`https://open-meteo.com/en/docs`). Grounded against the live source, not model memory.

### Seam 2: Serilog → rolling log file
- **(a) class:** `persistent-on-disk-state` (+ `host-OS` for the path & directory creation), **external** (the host filesystem).
- **(b) sides:** the app's Serilog logger (`OpenMeteoWeatherProvider` / bootstrap) ↔ the Windows filesystem under `%LOCALAPPDATA%`.
- **(c) contract:** Serilog is configured with a rolling **file** sink rooted at
  `%LOCALAPPDATA%\SimpleDesktopWeatherApp\logs\`; the directory is created if absent (host-OS
  path contract — must not throw if the folder does not yet exist). A successful fetch emits one
  INFO event carrying the request URL, HTTP status, and latency; a failed fetch emits one ERROR
  event carrying the exception and stack trace. No log content is ever surfaced in the UI.
  Nullability: the log directory path resolves from `%LOCALAPPDATA%` (always set on Windows); if
  unresolvable the app falls back to a relative `logs\` folder rather than crashing.
- **(d) proof:** Tier-1 — the provider is given a captured/in-memory logger; a fetch asserts an
  INFO event with URL+status+latency is emitted, and an induced failure asserts an ERROR event
  with the exception. The `%LOCALAPPDATA%` path resolution & directory creation is an observable
  verified at Tier-3 smoke (the log file appears on disk after a real run).

### Seam 3: appsettings.json → LocationOptions binding
- **(a) class:** `persistent-on-disk-state`, **internal** (we author the file).
- **(b) sides:** `appsettings.json` (shipped with the app, read at startup) ↔ `LocationOptions` via `Microsoft.Extensions.Configuration` binding.
- **(c) contract:** `appsettings.json` carries a `"Location"` section binding to
  `LocationOptions { double Latitude; double Longitude; string Label; }`. Seed values:
  `Latitude = 51.51`, `Longitude = -0.13`, `Label = "London"`. Nullability/validation: if the
  section is absent or a field is missing, the binder yields defaults (`0, 0, null`); startup
  **validates** the bound options (non-null/non-empty `Label`, coordinates in valid lat/lon range)
  and **fails fast with a clear log message** rather than letting a null `Label` or a `(0,0)`
  coordinate reach the provider. This is the single seam Feature 2's Place Search replaces.
- **(d) proof:** Tier-1 — a sample config section binds to the expected `LocationOptions`
  (round-trip), and a missing/empty section trips the validation error rather than producing a
  silent `(0,0,null)`.
