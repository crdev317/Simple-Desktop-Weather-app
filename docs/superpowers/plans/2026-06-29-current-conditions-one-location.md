# Show Current Conditions for one location — Implementation Plan

> **For agentic workers:** Do NOT implement this plan directly. It must first pass `/feature-doc-gauntlet` in a clean session, then be broken into stories by `/enate-to-stories`; AFK implementation happens per-story from there. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On launch, fetch and display the Current Conditions (temperature °C + readable Condition + observation time) for a single hard-coded Location (London) from Open-Meteo, in a WPF window that never crashes on a failed or slow fetch.

**Architecture:** Three-project .NET 10 solution — `WeatherApp.Core` (domain, Open-Meteo provider, Condition Mapper, view-model; no WPF reference), `WeatherApp` (WPF Views + generic-Host/DI bootstrap), `WeatherApp.Tests` (references Core only). MVVM with `CommunityToolkit.Mvvm`; HTTP via `IHttpClientFactory` + `System.Text.Json`; logging via Serilog to a rolling file. The view-model is the single choke point that collapses every fetch failure into observable `Failed` state.

**Tech Stack:** .NET 10 / WPF (C#); `CommunityToolkit.Mvvm`; `Microsoft.Extensions.Hosting` / `.DependencyInjection` / `.Http`; `System.Net.Http.Json` + `System.Text.Json`; `Serilog` + `Serilog.Sinks.File` + `Serilog.Extensions.Hosting`; tests with `xUnit` + `FluentAssertions` + `WireMock.Net`. (All from `Technical-Context.MD` → Packages in use — no new dependencies.)

**Context references:**
- Spec: `docs/superpowers/specs/2026-06-29-current-conditions-one-location-design.md`
- `business-domain-context.md` (the project's domain glossary / Context.MD)
- `Technical-Context.MD` (Overriding Principles that apply: **#3** UI never crashes on a failed/slow fetch; **#4** all network I/O async + cancellable; **#1** secrets via Credential Manager — not engaged, Open-Meteo is keyless; **#2** no breaking public-contract change without an ADR — the keyless-tier choice is ADR-gated)
- ADRs: none (the spec referenced an empty `docs/adr/`)

> An AFK Developer Agent picking up this plan MUST load every file in the Context references block before writing code.

---

## File structure

```
SimpleDesktopWeatherApp.sln
src/
  WeatherApp.Core/
    WeatherApp.Core.csproj
    Coordinates.cs                 — lat/lon value object (Location identity)
    Reading.cs                     — observed snapshot (temp °C, WMO code, time)
    Condition.cs                   — display Condition (text + reserved icon key)
    ConditionMapper.cs             — pure WMO code → Condition (total, fallback)
    IWeatherProvider.cs            — provider port
    OpenMeteoWeatherProvider.cs    — Open-Meteo HTTP client + JSON→Reading + logging
    LocationOptions.cs             — hard-coded Location, bound from config
    ViewModelStatus.cs             — Loading | Loaded | Failed
    CurrentConditionsViewModel.cs  — orchestration + Failed-on-error spine
  WeatherApp/
    WeatherApp.csproj
    App.xaml / App.xaml.cs         — generic Host, DI wiring, Serilog, show MainWindow
    MainWindow.xaml / .xaml.cs     — binds the view-model; renders the 3 states
    appsettings.json               — the "Location" section (London)
tests/
  WeatherApp.Tests/
    WeatherApp.Tests.csproj
    Fixtures/open-meteo-london-current.json   — captured live payload (Seam 1 (d))
    ConditionMapperTests.cs
    OpenMeteoWeatherProviderTests.cs          — Tier 1 (WireMock) + Tier 2 (live, trait-gated)
    LocationOptionsBindingTests.cs
    CurrentConditionsViewModelTests.cs
    LoggingTests.cs                           — Seam 2 Tier-1: file-sink on-disk proof + in-process diagnostics check
```

**Seam → Task coverage map** (verified in self-review):
- **Seam 1 (Open-Meteo current-weather HTTP, external):** Task 4 (Tier-1 WireMock replay of the captured fixture + malformed-body case = (d)) and Task 9 (Tier-2 live call = (d) against the real service).
- **Seam 2 (Serilog → rolling log file):** Task 8 — two proofs for the two facets: a **real Serilog file-sink boundary-crossing test** writing to a temp dir and asserting the file on disk + directory creation = (d) for the on-disk/host-OS channel (c-ii); plus a captured-logger unit check of INFO-on-success / ERROR-on-failure for the in-process diagnostics facet (c-i). The `%LOCALAPPDATA%`-specific path is a Tier-3 smoke observable.
- **Seam 3 (appsettings.json → LocationOptions, internal):** Task 6 (binding round-trip + missing-section validation = (d)).

---

## Task 1: Solution and project scaffold

**Files:**
- Create: `SimpleDesktopWeatherApp.sln`
- Create: `src/WeatherApp.Core/WeatherApp.Core.csproj`
- Create: `src/WeatherApp/WeatherApp.csproj`
- Create: `tests/WeatherApp.Tests/WeatherApp.Tests.csproj`

- [ ] **Step 1: Create the solution and three projects**

```bash
dotnet new sln -n SimpleDesktopWeatherApp
dotnet new classlib -n WeatherApp.Core -o src/WeatherApp.Core -f net10.0
dotnet new wpf      -n WeatherApp      -o src/WeatherApp      -f net10.0-windows
dotnet new xunit    -n WeatherApp.Tests -o tests/WeatherApp.Tests -f net10.0
rm src/WeatherApp.Core/Class1.cs tests/WeatherApp.Tests/UnitTest1.cs
dotnet sln add src/WeatherApp.Core src/WeatherApp tests/WeatherApp.Tests
```

- [ ] **Step 2: Wire project references (enforce the Core/UI boundary)**

```bash
dotnet add src/WeatherApp        reference src/WeatherApp.Core
dotnet add tests/WeatherApp.Tests reference src/WeatherApp.Core
# NOTE: Tests does NOT reference WeatherApp (WPF). A view-model touching System.Windows
# would then fail to compile in Tests — the build-enforced boundary from the spec.
```

- [ ] **Step 3: Add packages (only those in Technical-Context.MD → Packages in use)**

```bash
# Core
dotnet add src/WeatherApp.Core package CommunityToolkit.Mvvm
dotnet add src/WeatherApp.Core package Microsoft.Extensions.Http
dotnet add src/WeatherApp.Core package Microsoft.Extensions.Options
dotnet add src/WeatherApp.Core package System.Net.Http.Json
dotnet add src/WeatherApp.Core package Microsoft.Extensions.Logging.Abstractions
# WPF app
dotnet add src/WeatherApp package Microsoft.Extensions.Hosting
dotnet add src/WeatherApp package Serilog.Extensions.Hosting
dotnet add src/WeatherApp package Serilog.Sinks.File
# Tests
dotnet add tests/WeatherApp.Tests package FluentAssertions
dotnet add tests/WeatherApp.Tests package WireMock.Net
dotnet add tests/WeatherApp.Tests package Microsoft.Extensions.Configuration.Json
# Serilog — needed by Task 8's on-disk file-sink boundary-crossing test (Seam 2 (c-ii) proof)
dotnet add tests/WeatherApp.Tests package Serilog
dotnet add tests/WeatherApp.Tests package Serilog.Sinks.File
dotnet add tests/WeatherApp.Tests package Serilog.Extensions.Logging
```

- [ ] **Step 4: Set `Nullable` + `ImplicitUsings` enabled in every csproj**

Ensure each `.csproj` `<PropertyGroup>` contains:

```xml
<Nullable>enable</Nullable>
<ImplicitUsings>enable</ImplicitUsings>
```

- [ ] **Step 5: Verify the solution builds empty**

Run: `dotnet build`
Expected: Build succeeded, 0 warnings, 0 errors.

- [ ] **Step 6: Commit**

```bash
git add SimpleDesktopWeatherApp.sln src/ tests/
git commit -m "chore: scaffold 3-project solution (Core/WPF/Tests)"
```

---

## Task 2: Domain types — `Coordinates`, `Reading`, `Condition`

**Files:**
- Create: `src/WeatherApp.Core/Coordinates.cs`
- Create: `src/WeatherApp.Core/Condition.cs`
- Create: `src/WeatherApp.Core/Reading.cs`

These are pure immutable data with no behaviour, so they are introduced as plain records; the first test that exercises them is the Condition Mapper test in Task 3.

- [ ] **Step 1: Create `Coordinates`**

```csharp
namespace WeatherApp.Core;

/// <summary>A geographic point. A Location's identity is its coordinates (per the glossary).</summary>
public readonly record struct Coordinates(double Latitude, double Longitude);
```

- [ ] **Step 2: Create `Condition`**

```csharp
namespace WeatherApp.Core;

/// <summary>A display Condition: readable text plus a stable icon key (icon unused in v1).</summary>
public readonly record struct Condition(string Text, string IconKey);
```

- [ ] **Step 3: Create `Reading`**

```csharp
namespace WeatherApp.Core;

/// <summary>
/// A single snapshot of observed weather for a Location (Current Conditions = the present Reading).
/// Never a prediction.
/// </summary>
/// <param name="TemperatureCelsius">Observed air temperature in °C.</param>
/// <param name="WmoCode">Open-Meteo WMO weather code.</param>
/// <param name="Condition">Display Condition mapped from <paramref name="WmoCode"/>.</param>
/// <param name="ObservedAt">The Reading's observation time as reported by the provider.</param>
public sealed record Reading(
    double TemperatureCelsius,
    int WmoCode,
    Condition Condition,
    DateTimeOffset ObservedAt);
```

- [ ] **Step 4: Build**

Run: `dotnet build src/WeatherApp.Core`
Expected: Build succeeded.

- [ ] **Step 5: Commit**

```bash
git add src/WeatherApp.Core/Coordinates.cs src/WeatherApp.Core/Condition.cs src/WeatherApp.Core/Reading.cs
git commit -m "feat(core): add Coordinates, Reading, Condition domain types"
```

---

## Task 3: Condition Mapper (pure, table-driven — Tier 1)

**Files:**
- Create: `src/WeatherApp.Core/ConditionMapper.cs`
- Test: `tests/WeatherApp.Tests/ConditionMapperTests.cs`

The Open-Meteo `weather_code` is a WMO code. The mapper must be **total** — every input returns a `Condition`, with an explicit fallback for unknown codes, and must never throw.

- [ ] **Step 1: Write the failing tests**

```csharp
using FluentAssertions;
using WeatherApp.Core;
using Xunit;

namespace WeatherApp.Tests;

public class ConditionMapperTests
{
    [Theory]
    [InlineData(0, "Clear sky")]
    [InlineData(2, "Partly cloudy")]
    [InlineData(3, "Overcast")]   // the value the live London call returned
    [InlineData(61, "Rain")]
    [InlineData(95, "Thunderstorm")]
    public void Maps_known_wmo_code_to_expected_text(int code, string expectedText)
    {
        ConditionMapper.Map(code).Text.Should().Be(expectedText);
    }

    [Fact]
    public void Unknown_code_returns_explicit_fallback_without_throwing()
    {
        var condition = ConditionMapper.Map(999);
        condition.Text.Should().Be("Unknown");
        condition.IconKey.Should().Be("unknown");
    }

    [Fact]
    public void Every_code_in_0_to_99_returns_a_non_empty_condition()
    {
        for (var code = 0; code <= 99; code++)
        {
            var c = ConditionMapper.Map(code);
            c.Text.Should().NotBeNullOrWhiteSpace();
            c.IconKey.Should().NotBeNullOrWhiteSpace();
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `dotnet test --filter ConditionMapperTests`
Expected: FAIL — `ConditionMapper` does not exist.

- [ ] **Step 3: Write minimal implementation**

```csharp
namespace WeatherApp.Core;

/// <summary>Pure mapping of an Open-Meteo WMO weather code to a display <see cref="Condition"/>.</summary>
public static class ConditionMapper
{
    private static readonly Condition Unknown = new("Unknown", "unknown");

    // WMO 4677 codes used by Open-Meteo. Grouped to the text the UI shows in v1.
    private static readonly IReadOnlyDictionary<int, Condition> Map_ = new Dictionary<int, Condition>
    {
        [0] = new("Clear sky", "clear"),
        [1] = new("Mainly clear", "mostly-clear"),
        [2] = new("Partly cloudy", "partly-cloudy"),
        [3] = new("Overcast", "overcast"),
        [45] = new("Fog", "fog"),
        [48] = new("Depositing rime fog", "fog"),
        [51] = new("Light drizzle", "drizzle"),
        [53] = new("Moderate drizzle", "drizzle"),
        [55] = new("Dense drizzle", "drizzle"),
        [56] = new("Light freezing drizzle", "freezing-drizzle"),
        [57] = new("Dense freezing drizzle", "freezing-drizzle"),
        [61] = new("Rain", "rain"),
        [63] = new("Moderate rain", "rain"),
        [65] = new("Heavy rain", "rain"),
        [66] = new("Light freezing rain", "freezing-rain"),
        [67] = new("Heavy freezing rain", "freezing-rain"),
        [71] = new("Slight snow", "snow"),
        [73] = new("Moderate snow", "snow"),
        [75] = new("Heavy snow", "snow"),
        [77] = new("Snow grains", "snow"),
        [80] = new("Slight rain showers", "rain-showers"),
        [81] = new("Moderate rain showers", "rain-showers"),
        [82] = new("Violent rain showers", "rain-showers"),
        [85] = new("Slight snow showers", "snow-showers"),
        [86] = new("Heavy snow showers", "snow-showers"),
        [95] = new("Thunderstorm", "thunderstorm"),
        [96] = new("Thunderstorm with slight hail", "thunderstorm-hail"),
        [99] = new("Thunderstorm with heavy hail", "thunderstorm-hail"),
    };

    /// <summary>Returns the display Condition for a WMO code; never throws — unmapped codes return the explicit fallback.</summary>
    public static Condition Map(int wmoCode) => Map_.TryGetValue(wmoCode, out var c) ? c : Unknown;
}
```

> The `Every_code_in_0_to_99` test passes because unmapped codes fall through to `Unknown` (non-empty). This is the intended "total with fallback" contract, not a gap.

- [ ] **Step 4: Run tests to verify they pass**

Run: `dotnet test --filter ConditionMapperTests`
Expected: PASS (all theory cases + both facts).

- [ ] **Step 5: Commit**

```bash
git add src/WeatherApp.Core/ConditionMapper.cs tests/WeatherApp.Tests/ConditionMapperTests.cs
git commit -m "feat(core): add total WMO-code Condition Mapper with fallback"
```

---

## Task 4: Open-Meteo Weather Provider — parse contract (Seam 1, Tier 1)

**Files:**
- Create: `src/WeatherApp.Core/IWeatherProvider.cs`
- Create: `src/WeatherApp.Core/OpenMeteoWeatherProvider.cs`
- Create: `tests/WeatherApp.Tests/Fixtures/open-meteo-london-current.json`
- Test: `tests/WeatherApp.Tests/OpenMeteoWeatherProviderTests.cs`

> **Seam 1 (c) contract (verbatim from the spec):** `GET .../v1/forecast?latitude={lat}&longitude={lon}&current=temperature_2m,weather_code&timezone=GMT`; HTTP 200 JSON with a non-null `current` object holding `temperature_2m` (non-null number, °C), `weather_code` (non-null int, WMO), `time` (non-null ISO-8601 string); field names snake_case; non-2xx / timeout / missing-`current` ⇒ failed fetch; keyless auth.

- [ ] **Step 1: Save the captured live payload as the Tier-1 fixture (Seam 1 (d))**

Create `tests/WeatherApp.Tests/Fixtures/open-meteo-london-current.json` with the exact body observed during brainstorming:

```json
{"latitude":51.5,"longitude":-0.25,"generationtime_ms":0.35,"utc_offset_seconds":0,"timezone":"GMT","timezone_abbreviation":"GMT","elevation":29.0,"current_units":{"time":"iso8601","interval":"seconds","temperature_2m":"°C","weather_code":"wmo code"},"current":{"time":"2026-06-29T14:15","interval":900,"temperature_2m":23.6,"weather_code":3}}
```

Mark it copy-to-output in `WeatherApp.Tests.csproj`:

```xml
<ItemGroup>
  <None Update="Fixtures/open-meteo-london-current.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
</ItemGroup>
```

- [ ] **Step 2: Define the port `IWeatherProvider`**

```csharp
namespace WeatherApp.Core;

/// <summary>Fetches the Current Conditions (present Reading) for a Location.</summary>
public interface IWeatherProvider
{
    /// <summary>
    /// Returns the Current Conditions for <paramref name="coordinates"/>.
    /// Async and cancellable (Overriding Principle #4). Throws on any fetch/parse failure —
    /// the caller (view-model) is responsible for catching and degrading (Principle #3).
    /// </summary>
    Task<Reading> GetCurrentConditionsAsync(Coordinates coordinates, CancellationToken cancellationToken);
}
```

- [ ] **Step 3: Write the failing Tier-1 tests (WireMock = real HTTP on one side)**

```csharp
using System.Net;
using FluentAssertions;
using Microsoft.Extensions.Logging.Abstractions;
using WeatherApp.Core;
using WireMock.RequestBuilders;
using WireMock.ResponseBuilders;
using WireMock.Server;
using Xunit;

namespace WeatherApp.Tests;

public class OpenMeteoWeatherProviderTests : IDisposable
{
    private readonly WireMockServer _server = WireMockServer.Start();

    private OpenMeteoWeatherProvider CreateProvider()
    {
        var http = new HttpClient { BaseAddress = new Uri(_server.Urls[0]) };
        return new OpenMeteoWeatherProvider(http, NullLogger<OpenMeteoWeatherProvider>.Instance);
    }

    [Fact]
    public async Task Parses_captured_payload_into_expected_Reading()
    {
        var body = File.ReadAllText("Fixtures/open-meteo-london-current.json");
        _server.Given(Request.Create().WithPath("/v1/forecast").UsingGet())
               .RespondWith(Response.Create().WithStatusCode(200)
                   .WithHeader("Content-Type", "application/json").WithBody(body));

        var reading = await CreateProvider()
            .GetCurrentConditionsAsync(new Coordinates(51.51, -0.13), CancellationToken.None);

        reading.TemperatureCelsius.Should().Be(23.6);
        reading.WmoCode.Should().Be(3);
        reading.Condition.Text.Should().Be("Overcast");          // mapped via ConditionMapper
        reading.ObservedAt.Should().Be(DateTimeOffset.Parse("2026-06-29T14:15:00Z"));
    }

    [Fact]
    public async Task Sends_the_required_query_parameters()
    {
        var body = File.ReadAllText("Fixtures/open-meteo-london-current.json");
        _server.Given(Request.Create().WithPath("/v1/forecast").UsingGet())
               .RespondWith(Response.Create().WithStatusCode(200).WithBody(body));

        await CreateProvider().GetCurrentConditionsAsync(new Coordinates(51.51, -0.13), CancellationToken.None);

        var request = _server.LogEntries.Single().RequestMessage;
        request.Query!["latitude"].Should().Contain("51.51");
        request.Query!["longitude"].Should().Contain("-0.13");
        request.Query!["current"].Should().Contain("temperature_2m,weather_code");
        request.Query!["timezone"].Should().Contain("GMT");
    }

    [Fact]
    public async Task Non_success_status_throws_HttpRequestException()
    {
        _server.Given(Request.Create().WithPath("/v1/forecast").UsingGet())
               .RespondWith(Response.Create().WithStatusCode(HttpStatusCode.BadRequest)
                   .WithBody("{\"error\":true,\"reason\":\"bad parameter\"}"));

        var act = () => CreateProvider()
            .GetCurrentConditionsAsync(new Coordinates(0, 0), CancellationToken.None);

        await act.Should().ThrowAsync<HttpRequestException>();
    }

    [Fact]
    public async Task Body_missing_current_object_throws()
    {
        _server.Given(Request.Create().WithPath("/v1/forecast").UsingGet())
               .RespondWith(Response.Create().WithStatusCode(200).WithBody("{\"latitude\":51.5}"));

        var act = () => CreateProvider()
            .GetCurrentConditionsAsync(new Coordinates(51.51, -0.13), CancellationToken.None);

        await act.Should().ThrowAsync<Exception>();   // missing required 'current' ⇒ failed fetch
    }

    public void Dispose() => _server.Dispose();
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `dotnet test --filter OpenMeteoWeatherProviderTests`
Expected: FAIL — `OpenMeteoWeatherProvider` does not exist.

- [ ] **Step 5: Implement the provider (parse only; logging added in Task 8)**

```csharp
using System.Net.Http.Json;
using System.Text.Json;
using System.Text.Json.Serialization;
using Microsoft.Extensions.Logging;

namespace WeatherApp.Core;

/// <summary>Open-Meteo implementation of <see cref="IWeatherProvider"/> (keyless free tier).</summary>
public sealed class OpenMeteoWeatherProvider : IWeatherProvider
{
    private readonly HttpClient _http;
    private readonly ILogger<OpenMeteoWeatherProvider> _logger;

    public OpenMeteoWeatherProvider(HttpClient http, ILogger<OpenMeteoWeatherProvider> logger)
    {
        _http = http;
        _logger = logger;
    }

    public async Task<Reading> GetCurrentConditionsAsync(Coordinates coordinates, CancellationToken cancellationToken)
    {
        var url = $"v1/forecast?latitude={coordinates.Latitude}&longitude={coordinates.Longitude}"
                + "&current=temperature_2m,weather_code&timezone=GMT";

        using var response = await _http.GetAsync(url, cancellationToken);
        response.EnsureSuccessStatusCode();   // non-2xx ⇒ HttpRequestException

        var dto = await response.Content.ReadFromJsonAsync<ForecastResponse>(cancellationToken);
        if (dto?.Current is null)
            throw new JsonException("Open-Meteo response missing required 'current' object.");

        var current = dto.Current;
        return new Reading(
            TemperatureCelsius: current.Temperature,
            WmoCode: current.WeatherCode,
            Condition: ConditionMapper.Map(current.WeatherCode),
            ObservedAt: new DateTimeOffset(DateTime.Parse(current.Time), TimeSpan.Zero));
    }

    // Private DTOs — the wire shape, snake_case mapped explicitly (Seam 1 (c)).
    private sealed record ForecastResponse([property: JsonPropertyName("current")] CurrentDto? Current);

    private sealed record CurrentDto(
        [property: JsonPropertyName("time")] string Time,
        [property: JsonPropertyName("temperature_2m")] double Temperature,
        [property: JsonPropertyName("weather_code")] int WeatherCode);
}
```

> `timezone=GMT` ⇒ `current.time` is UTC, so `TimeSpan.Zero` is correct here. If a later Feature changes `timezone`, the offset handling moves with it.

- [ ] **Step 6: Run tests to verify they pass**

Run: `dotnet test --filter OpenMeteoWeatherProviderTests`
Expected: PASS (parse, query params, non-success throws, missing-`current` throws).

- [ ] **Step 7: Commit**

```bash
git add src/WeatherApp.Core/IWeatherProvider.cs src/WeatherApp.Core/OpenMeteoWeatherProvider.cs \
        tests/WeatherApp.Tests/Fixtures/open-meteo-london-current.json \
        tests/WeatherApp.Tests/OpenMeteoWeatherProviderTests.cs tests/WeatherApp.Tests/WeatherApp.Tests.csproj
git commit -m "feat(core): Open-Meteo provider parses current conditions into Reading (Seam 1 Tier-1)"
```

---

## Task 5: View-model status enum

**Files:**
- Create: `src/WeatherApp.Core/ViewModelStatus.cs`

- [ ] **Step 1: Create the status enum**

```csharp
namespace WeatherApp.Core;

/// <summary>The view-model's display state. Drives which section MainWindow shows.</summary>
public enum ViewModelStatus
{
    Loading,
    Loaded,
    Failed
}
```

- [ ] **Step 2: Build**

Run: `dotnet build src/WeatherApp.Core`
Expected: Build succeeded.

- [ ] **Step 3: Commit**

```bash
git add src/WeatherApp.Core/ViewModelStatus.cs
git commit -m "feat(core): add ViewModelStatus enum"
```

---

## Task 6: `LocationOptions` + config binding (Seam 3, Tier 1)

**Files:**
- Create: `src/WeatherApp.Core/LocationOptions.cs`
- Test: `tests/WeatherApp.Tests/LocationOptionsBindingTests.cs`

> **Seam 3 (c) contract:** `appsettings.json` carries a `"Location"` section binding to `LocationOptions { double Latitude; double Longitude; string Label; }`; a missing section / field must fail fast with a clear message rather than yielding a silent `(0,0,null)`.

- [ ] **Step 1: Write the failing tests**

```csharp
using FluentAssertions;
using Microsoft.Extensions.Configuration;
using WeatherApp.Core;
using Xunit;

namespace WeatherApp.Tests;

public class LocationOptionsBindingTests
{
    private static IConfiguration Config(string json)
    {
        var path = Path.GetTempFileName();
        File.WriteAllText(path, json);
        return new ConfigurationBuilder().AddJsonFile(path).Build();
    }

    [Fact]
    public void Binds_Location_section_to_options()
    {
        var config = Config("""
            { "Location": { "Latitude": 51.51, "Longitude": -0.13, "Label": "London" } }
            """);

        var options = LocationOptions.FromConfiguration(config);

        options.Latitude.Should().Be(51.51);
        options.Longitude.Should().Be(-0.13);
        options.Label.Should().Be("London");
        options.ToCoordinates().Should().Be(new Coordinates(51.51, -0.13));
    }

    [Fact]
    public void Missing_section_fails_fast_with_clear_message()
    {
        var config = Config("""{ "SomethingElse": true }""");

        var act = () => LocationOptions.FromConfiguration(config);

        act.Should().Throw<InvalidOperationException>()
           .WithMessage("*Location*");   // names the missing section, not a null-deref later
    }

    [Fact]
    public void Empty_label_fails_validation()
    {
        var config = Config("""
            { "Location": { "Latitude": 51.51, "Longitude": -0.13, "Label": "" } }
            """);

        var act = () => LocationOptions.FromConfiguration(config);

        act.Should().Throw<InvalidOperationException>().WithMessage("*Label*");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `dotnet test --filter LocationOptionsBindingTests`
Expected: FAIL — `LocationOptions` does not exist.

- [ ] **Step 3: Implement `LocationOptions` with fail-fast binding**

```csharp
using Microsoft.Extensions.Configuration;

namespace WeatherApp.Core;

/// <summary>The single hard-coded Location for Feature 1, bound from the "Location" config section.
/// This is the seam Feature 2's Place Search replaces.</summary>
public sealed class LocationOptions
{
    public const string SectionName = "Location";

    public double Latitude { get; init; }
    public double Longitude { get; init; }
    public string Label { get; init; } = string.Empty;

    public Coordinates ToCoordinates() => new(Latitude, Longitude);

    /// <summary>Binds and validates the "Location" section; throws a clear error if absent or invalid.</summary>
    public static LocationOptions FromConfiguration(IConfiguration configuration)
    {
        var section = configuration.GetSection(SectionName);
        if (!section.Exists())
            throw new InvalidOperationException(
                $"Configuration is missing the required '{SectionName}' section.");

        var options = section.Get<LocationOptions>()
            ?? throw new InvalidOperationException($"Could not bind the '{SectionName}' section.");

        if (string.IsNullOrWhiteSpace(options.Label))
            throw new InvalidOperationException($"'{SectionName}:Label' must be a non-empty string.");
        if (options.Latitude is < -90 or > 90)
            throw new InvalidOperationException($"'{SectionName}:Latitude' must be between -90 and 90.");
        if (options.Longitude is < -180 or > 180)
            throw new InvalidOperationException($"'{SectionName}:Longitude' must be between -180 and 180.");

        return options;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `dotnet test --filter LocationOptionsBindingTests`
Expected: PASS (bind, missing-section, empty-label).

- [ ] **Step 5: Commit**

```bash
git add src/WeatherApp.Core/LocationOptions.cs tests/WeatherApp.Tests/LocationOptionsBindingTests.cs
git commit -m "feat(core): add LocationOptions with fail-fast config binding (Seam 3)"
```

---

## Task 7: `CurrentConditionsViewModel` — the Failed-on-error spine (Principle #3, Tier 1)

**Files:**
- Create: `src/WeatherApp.Core/CurrentConditionsViewModel.cs`
- Test: `tests/WeatherApp.Tests/CurrentConditionsViewModelTests.cs`

> This task pins the in-process failure contract from the spec: the provider may throw; the view-model catches **everything** and yields `Failed`, never letting an exception escape (Overriding Principle #3).

- [ ] **Step 1: Write the failing tests (off the UI thread, fake provider)**

```csharp
using FluentAssertions;
using WeatherApp.Core;
using Xunit;

namespace WeatherApp.Tests;

public class CurrentConditionsViewModelTests
{
    private static readonly LocationOptions London =
        new() { Latitude = 51.51, Longitude = -0.13, Label = "London" };

    private sealed class FakeProvider : IWeatherProvider
    {
        private readonly Func<Reading> _behaviour;
        public FakeProvider(Func<Reading> behaviour) => _behaviour = behaviour;
        public Task<Reading> GetCurrentConditionsAsync(Coordinates c, CancellationToken ct)
        {
            ct.ThrowIfCancellationRequested();
            return Task.FromResult(_behaviour());
        }
    }

    [Fact]
    public async Task Successful_fetch_sets_Loaded_and_display_strings()
    {
        var reading = new Reading(23.6, 3, new Condition("Overcast", "overcast"),
            new DateTimeOffset(2026, 6, 29, 14, 15, 0, TimeSpan.Zero));
        var vm = new CurrentConditionsViewModel(new FakeProvider(() => reading), London);

        await vm.LoadAsync(CancellationToken.None);

        vm.Status.Should().Be(ViewModelStatus.Loaded);
        vm.LocationLabel.Should().Be("London");
        vm.Temperature.Should().Be("24°C");          // rounded to whole degrees for display
        vm.ConditionText.Should().Be("Overcast");
        vm.ObservedAtText.Should().Be("as of 14:15");
    }

    [Fact]
    public async Task Provider_exception_sets_Failed_and_does_not_throw()
    {
        var vm = new CurrentConditionsViewModel(
            new FakeProvider(() => throw new HttpRequestException("boom")), London);

        await vm.LoadAsync(CancellationToken.None);   // must not throw

        vm.Status.Should().Be(ViewModelStatus.Failed);
        vm.ErrorMessage.Should().Be("Couldn't load the weather.");
    }

    [Fact]
    public async Task Cancellation_sets_Failed_and_does_not_throw()
    {
        var vm = new CurrentConditionsViewModel(
            new FakeProvider(() => throw new InvalidOperationException()), London);
        using var cts = new CancellationTokenSource();
        cts.Cancel();

        await vm.LoadAsync(cts.Token);   // must not throw

        vm.Status.Should().Be(ViewModelStatus.Failed);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `dotnet test --filter CurrentConditionsViewModelTests`
Expected: FAIL — `CurrentConditionsViewModel` does not exist.

- [ ] **Step 3: Implement the view-model**

```csharp
using System.Globalization;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace WeatherApp.Core;

/// <summary>Orchestrates the launch fetch for the hard-coded Location and exposes display state.
/// The single choke point that collapses every fetch failure into <see cref="ViewModelStatus.Failed"/>
/// (Overriding Principle #3). No UI-thread dependencies.</summary>
public sealed partial class CurrentConditionsViewModel : ObservableObject
{
    private readonly IWeatherProvider _provider;
    private readonly LocationOptions _location;

    public CurrentConditionsViewModel(IWeatherProvider provider, LocationOptions location)
    {
        _provider = provider;
        _location = location;
        LocationLabel = location.Label;
    }

    [ObservableProperty] private ViewModelStatus _status = ViewModelStatus.Loading;
    [ObservableProperty] private string _locationLabel;
    [ObservableProperty] private string _temperature = string.Empty;
    [ObservableProperty] private string _conditionText = string.Empty;
    [ObservableProperty] private string _observedAtText = string.Empty;
    [ObservableProperty] private string _errorMessage = string.Empty;

    [RelayCommand]
    public async Task LoadAsync(CancellationToken cancellationToken)
    {
        Status = ViewModelStatus.Loading;
        try
        {
            var reading = await _provider.GetCurrentConditionsAsync(
                _location.ToCoordinates(), cancellationToken);

            Temperature = $"{Math.Round(reading.TemperatureCelsius, MidpointRounding.AwayFromZero):0}°C";
            ConditionText = reading.Condition.Text;
            ObservedAtText = $"as of {reading.ObservedAt.ToString("HH:mm", CultureInfo.InvariantCulture)}";
            Status = ViewModelStatus.Loaded;
        }
        catch (Exception)   // Principle #3: never let any fetch/parse/cancellation failure reach the UI
        {
            ErrorMessage = "Couldn't load the weather.";
            Status = ViewModelStatus.Failed;
        }
    }
}
```

> `[ObservableProperty]` on `_temperature` generates the public `Temperature` property — names in the tests (`Temperature`, `ConditionText`, etc.) match the generated members. The blanket `catch (Exception)` is deliberate and is the explicit expression of Principle #3 for the tracer.

- [ ] **Step 4: Run tests to verify they pass**

Run: `dotnet test --filter CurrentConditionsViewModelTests`
Expected: PASS (Loaded + strings; exception ⇒ Failed; cancellation ⇒ Failed).

- [ ] **Step 5: Commit**

```bash
git add src/WeatherApp.Core/CurrentConditionsViewModel.cs tests/WeatherApp.Tests/CurrentConditionsViewModelTests.cs
git commit -m "feat(core): CurrentConditionsViewModel with Failed-on-error spine (Principle #3)"
```

---

## Task 8: Provider logging (Seam 2, Tier 1)

**Files:**
- Modify: `src/WeatherApp.Core/OpenMeteoWeatherProvider.cs`
- Test: `tests/WeatherApp.Tests/LoggingTests.cs`

> **Seam 2 (c) contract — two facets:** **(c-i) in-process diagnostics** — a successful fetch emits one INFO event (request URL + HTTP status + latency); a failed fetch emits one ERROR event (the exception). **(c-ii) on-disk / host-OS** (the declared channel) — Serilog's file sink **writes those events as lines to a file on disk under the configured log directory, creating the directory if absent**. Both facets are proven at Tier-1 below; the `%LOCALAPPDATA%`-specific path stays a Tier-3 smoke observable.

- [ ] **Step 1: Write the failing tests — the in-process diagnostics check AND the on-disk file-sink boundary-crossing test**

The `CapturingLogger` tests pin (c-i). The file-sink test pins (c-ii) by writing through a real Serilog `File` sink to a fresh temp directory and reading the bytes back off disk — real I/O on the filesystem side, so the proof crosses the channel the seam is classed for (it does **not** mock the sink).

```csharp
using FluentAssertions;
using Microsoft.Extensions.Logging;
using WeatherApp.Core;
using WireMock.RequestBuilders;
using WireMock.ResponseBuilders;
using WireMock.Server;
using Xunit;

namespace WeatherApp.Tests;

public class LoggingTests : IDisposable
{
    private readonly WireMockServer _server = WireMockServer.Start();

    private sealed record LogLine(LogLevel Level, string Message);

    private sealed class CapturingLogger<T> : ILogger<T>
    {
        public List<LogLine> Lines { get; } = new();
        public IDisposable BeginScope<TState>(TState state) where TState : notnull => NullScope.Instance;
        public bool IsEnabled(LogLevel logLevel) => true;
        public void Log<TState>(LogLevel level, EventId id, TState state, Exception? ex,
            Func<TState, Exception?, string> formatter)
            => Lines.Add(new LogLine(level, formatter(state, ex)));

        private sealed class NullScope : IDisposable
        { public static readonly NullScope Instance = new(); public void Dispose() { } }
    }

    [Fact]
    public async Task Successful_fetch_logs_Information_with_status_and_latency()
    {
        var body = File.ReadAllText("Fixtures/open-meteo-london-current.json");
        _server.Given(Request.Create().WithPath("/v1/forecast").UsingGet())
               .RespondWith(Response.Create().WithStatusCode(200).WithBody(body));
        var logger = new CapturingLogger<OpenMeteoWeatherProvider>();
        var http = new HttpClient { BaseAddress = new Uri(_server.Urls[0]) };

        await new OpenMeteoWeatherProvider(http, logger)
            .GetCurrentConditionsAsync(new Coordinates(51.51, -0.13), CancellationToken.None);

        logger.Lines.Should().ContainSingle(l => l.Level == LogLevel.Information)
              .Which.Message.Should().Contain("200").And.Contain("ms");
    }

    [Fact]
    public async Task Failed_fetch_logs_Error()
    {
        _server.Given(Request.Create().WithPath("/v1/forecast").UsingGet())
               .RespondWith(Response.Create().WithStatusCode(500));
        var logger = new CapturingLogger<OpenMeteoWeatherProvider>();
        var http = new HttpClient { BaseAddress = new Uri(_server.Urls[0]) };

        var act = () => new OpenMeteoWeatherProvider(http, logger)
            .GetCurrentConditionsAsync(new Coordinates(51.51, -0.13), CancellationToken.None);

        await act.Should().ThrowAsync<HttpRequestException>();
        logger.Lines.Should().Contain(l => l.Level == LogLevel.Error);
    }

    // (c-ii) on-disk / host-OS: the real boundary-crossing proof — a live Serilog File sink, a
    // not-yet-existing temp directory, and the written bytes read back off disk. No mocked sink.
    [Fact]
    public async Task Successful_fetch_writes_a_log_line_to_a_real_file_and_creates_the_directory()
    {
        var body = File.ReadAllText("Fixtures/open-meteo-london-current.json");
        _server.Given(Request.Create().WithPath("/v1/forecast").UsingGet())
               .RespondWith(Response.Create().WithStatusCode(200).WithBody(body));

        // A directory that does NOT exist yet — proves the host-OS directory-creation facet.
        var logDir = Path.Combine(Path.GetTempPath(), "weatherapp-logtest-" + Guid.NewGuid().ToString("N"));
        var logPath = Path.Combine(logDir, "log.txt");
        try
        {
            using (var serilog = new Serilog.LoggerConfiguration()
                       .MinimumLevel.Information()
                       .WriteTo.File(logPath)
                       .CreateLogger())
            {
                var factory = new Serilog.Extensions.Logging.SerilogLoggerFactory(serilog);
                ILogger<OpenMeteoWeatherProvider> logger = factory.CreateLogger<OpenMeteoWeatherProvider>();
                var http = new HttpClient { BaseAddress = new Uri(_server.Urls[0]) };

                await new OpenMeteoWeatherProvider(http, logger)
                    .GetCurrentConditionsAsync(new Coordinates(51.51, -0.13), CancellationToken.None);
            }   // disposing the Serilog logger flushes the file sink to disk

            Directory.Exists(logDir).Should().BeTrue();          // host-OS: directory was created
            var written = File.ReadAllText(logPath);
            written.Should().Contain("v1/forecast").And.Contain("200");   // the event crossed to disk
        }
        finally
        {
            if (Directory.Exists(logDir)) Directory.Delete(logDir, recursive: true);
        }
    }

    public void Dispose() => _server.Dispose();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `dotnet test --filter LoggingTests`
Expected: FAIL — provider does not log yet, so neither the captured-logger lines nor the on-disk file content appear.

- [ ] **Step 3: Add logging + latency + exception path to the provider**

Replace `GetCurrentConditionsAsync` in `OpenMeteoWeatherProvider.cs` with:

```csharp
    public async Task<Reading> GetCurrentConditionsAsync(Coordinates coordinates, CancellationToken cancellationToken)
    {
        var url = $"v1/forecast?latitude={coordinates.Latitude}&longitude={coordinates.Longitude}"
                + "&current=temperature_2m,weather_code&timezone=GMT";
        var startTimestamp = System.Diagnostics.Stopwatch.GetTimestamp();
        try
        {
            using var response = await _http.GetAsync(url, cancellationToken);
            var elapsedMs = System.Diagnostics.Stopwatch.GetElapsedTime(startTimestamp).TotalMilliseconds;
            _logger.LogInformation("Open-Meteo GET {Url} -> {Status} in {ElapsedMs:0}ms",
                url, (int)response.StatusCode, elapsedMs);

            response.EnsureSuccessStatusCode();

            var dto = await response.Content.ReadFromJsonAsync<ForecastResponse>(cancellationToken);
            if (dto?.Current is null)
                throw new JsonException("Open-Meteo response missing required 'current' object.");

            var current = dto.Current;
            return new Reading(
                TemperatureCelsius: current.Temperature,
                WmoCode: current.WeatherCode,
                Condition: ConditionMapper.Map(current.WeatherCode),
                ObservedAt: new DateTimeOffset(DateTime.Parse(current.Time), TimeSpan.Zero));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Open-Meteo fetch failed for {Url}", url);
            throw;   // re-throw: the view-model owns degradation (Principle #3), the provider owns diagnostics
        }
    }
```

> Add `using Microsoft.Extensions.Logging;` if not already present. The INFO line is emitted **before** `EnsureSuccessStatusCode`, so a logged 500 still produces both the INFO (status) and the ERROR (exception) — matching the contract.

- [ ] **Step 4: Run tests to verify they pass**

Run: `dotnet test --filter "LoggingTests|OpenMeteoWeatherProviderTests"`
Expected: PASS (logging asserts + the Task-4 parse tests still green).

- [ ] **Step 5: Commit**

```bash
git add src/WeatherApp.Core/OpenMeteoWeatherProvider.cs tests/WeatherApp.Tests/LoggingTests.cs
git commit -m "feat(core): log every Open-Meteo request (INFO) and failure (ERROR) (Seam 2 Tier-1)"
```

---

## Task 9: Live Tier-2 test (Seam 1 (d), trait-gated off the PR gate)

**Files:**
- Modify: `tests/WeatherApp.Tests/OpenMeteoWeatherProviderTests.cs`

> Tier-2 proves the live seam the fixture cannot. Single keyless call to real Open-Meteo for London. Tagged so the everyday `dotnet test` (PR gate) excludes it; it runs scheduled.

- [ ] **Step 1: Add the trait-gated live test**

```csharp
public class OpenMeteoLiveTests
{
    [Trait("Tier", "Live")]
    [Fact]
    public async Task Live_London_call_returns_a_parseable_Reading()
    {
        var http = new HttpClient { BaseAddress = new Uri("https://api.open-meteo.com/") };
        var provider = new OpenMeteoWeatherProvider(http,
            Microsoft.Extensions.Logging.Abstractions.NullLogger<OpenMeteoWeatherProvider>.Instance);

        var reading = await provider.GetCurrentConditionsAsync(
            new Coordinates(51.51, -0.13), CancellationToken.None);

        reading.TemperatureCelsius.Should().BeInRange(-90, 60);   // a real, sane temperature
        reading.Condition.Text.Should().NotBeNullOrWhiteSpace();
    }
}
```

- [ ] **Step 2: Verify the PR-gate run excludes it**

Run: `dotnet test --filter "Tier!=Live"`
Expected: PASS, and the live test is **not** executed (count excludes it).

- [ ] **Step 3: Verify the live test passes when explicitly selected (manual/scheduled)**

Run: `dotnet test --filter "Tier=Live"`
Expected: PASS (requires network; this is the scheduled Tier-2 run).

- [ ] **Step 4: Commit**

```bash
git add tests/WeatherApp.Tests/OpenMeteoWeatherProviderTests.cs
git commit -m "test: add trait-gated Tier-2 live Open-Meteo call (Seam 1 (d))"
```

---

## Task 10: WPF bootstrap — generic Host, DI, Serilog, window wiring

**Files:**
- Create: `src/WeatherApp/appsettings.json`
- Modify: `src/WeatherApp/WeatherApp.csproj`
- Modify: `src/WeatherApp/App.xaml` / `src/WeatherApp/App.xaml.cs`

This task has no unit test (it is composition + UI host); it is exercised at Tier-3 smoke (Task 11).

- [ ] **Step 1: Create `appsettings.json` (the London "Location" section — Seam 3 seed)**

```json
{
  "Location": {
    "Latitude": 51.51,
    "Longitude": -0.13,
    "Label": "London"
  }
}
```

- [ ] **Step 2: Copy `appsettings.json` to output in `WeatherApp.csproj`**

```xml
<ItemGroup>
  <None Update="appsettings.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
</ItemGroup>
```

- [ ] **Step 3: Disable WPF's auto `StartupUri` (App owns startup via the Host)**

In `App.xaml`, remove any `StartupUri="MainWindow.xaml"` attribute so `OnStartup` controls window creation.

```xml
<Application x:Class="WeatherApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Application.Resources/>
</Application>
```

- [ ] **Step 4: Build the Host, DI, and Serilog in `App.xaml.cs`**

```csharp
using System.IO;
using System.Windows;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;
using WeatherApp.Core;

namespace WeatherApp;

public partial class App : Application
{
    private IHost? _host;

    protected override async void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        var logDirectory = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
            "SimpleDesktopWeatherApp", "logs");
        Directory.CreateDirectory(logDirectory);   // host-OS path contract (Seam 2)

        _host = Host.CreateDefaultBuilder()
            .UseSerilog((context, config) => config
                .MinimumLevel.Information()
                .WriteTo.File(Path.Combine(logDirectory, "log-.txt"), rollingInterval: RollingInterval.Day))
            .ConfigureServices((context, services) =>
            {
                services.AddSingleton(sp => LocationOptions.FromConfiguration(context.Configuration));
                services.AddHttpClient<IWeatherProvider, OpenMeteoWeatherProvider>(client =>
                {
                    client.BaseAddress = new Uri("https://api.open-meteo.com/");
                    client.Timeout = TimeSpan.FromSeconds(10);   // bounded — a hang becomes Failed, not a freeze
                });
                services.AddTransient<CurrentConditionsViewModel>();
                services.AddTransient<MainWindow>();
            })
            .Build();

        await _host.StartAsync();

        var window = _host.Services.GetRequiredService<MainWindow>();
        window.Show();
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        if (_host is not null)
        {
            await _host.StopAsync();
            _host.Dispose();
        }
        Log.CloseAndFlush();
        base.OnExit(e);
    }
}
```

> `AddHttpClient<IWeatherProvider, OpenMeteoWeatherProvider>` is the `IHttpClientFactory` typed-client registration from `Technical-Context.MD`; it injects a managed `HttpClient` into the provider. `Host.CreateDefaultBuilder` already loads `appsettings.json` into `context.Configuration`.

- [ ] **Step 5: Build**

Run: `dotnet build src/WeatherApp`
Expected: Build succeeded.

- [ ] **Step 6: Commit**

```bash
git add src/WeatherApp/appsettings.json src/WeatherApp/WeatherApp.csproj src/WeatherApp/App.xaml src/WeatherApp/App.xaml.cs
git commit -m "feat(app): generic Host + DI + Serilog rolling-file bootstrap"
```

---

## Task 11: MainWindow — bind the view-model and render the three states

**Files:**
- Modify: `src/WeatherApp/MainWindow.xaml` / `src/WeatherApp/MainWindow.xaml.cs`

- [ ] **Step 1: Constructor-inject the view-model and fire the launch fetch**

```csharp
using System.Windows;
using WeatherApp.Core;

namespace WeatherApp;

public partial class MainWindow : Window
{
    private readonly CurrentConditionsViewModel _viewModel;

    public MainWindow(CurrentConditionsViewModel viewModel)
    {
        InitializeComponent();
        _viewModel = viewModel;
        DataContext = _viewModel;
        Loaded += async (_, _) => await _viewModel.LoadAsync(CancellationToken.None);
    }
}
```

- [ ] **Step 2: Lay out the three states in `MainWindow.xaml`**

Visibility is driven by `Status` via a converter. Each state is a panel toggled by the bound status; the four display strings bind directly.

```xml
<Window x:Class="WeatherApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:core="clr-namespace:WeatherApp.Core;assembly=WeatherApp.Core"
        xmlns:local="clr-namespace:WeatherApp"
        Title="Weather" Height="240" Width="360">
    <Window.Resources>
        <local:StatusToVisibilityConverter x:Key="StatusToVisibility"/>
    </Window.Resources>
    <Grid Margin="24">
        <!-- Loading -->
        <TextBlock Text="Loading…" FontSize="16" HorizontalAlignment="Center" VerticalAlignment="Center"
                   Visibility="{Binding Status, Converter={StaticResource StatusToVisibility},
                                ConverterParameter={x:Static core:ViewModelStatus.Loading}}"/>
        <!-- Loaded -->
        <StackPanel VerticalAlignment="Center"
                    Visibility="{Binding Status, Converter={StaticResource StatusToVisibility},
                                 ConverterParameter={x:Static core:ViewModelStatus.Loaded}}">
            <TextBlock Text="{Binding LocationLabel}" FontSize="20" FontWeight="Bold" HorizontalAlignment="Center"/>
            <TextBlock Text="{Binding Temperature}" FontSize="40" HorizontalAlignment="Center"/>
            <TextBlock Text="{Binding ConditionText}" FontSize="16" HorizontalAlignment="Center"/>
            <TextBlock Text="{Binding ObservedAtText}" FontSize="12" Foreground="Gray" HorizontalAlignment="Center"/>
        </StackPanel>
        <!-- Failed -->
        <TextBlock Text="{Binding ErrorMessage}" FontSize="16" Foreground="#B00020"
                   HorizontalAlignment="Center" VerticalAlignment="Center" TextWrapping="Wrap"
                   Visibility="{Binding Status, Converter={StaticResource StatusToVisibility},
                                ConverterParameter={x:Static core:ViewModelStatus.Failed}}"/>
    </Grid>
</Window>
```

- [ ] **Step 3: Add the `StatusToVisibilityConverter`**

Create `src/WeatherApp/StatusToVisibilityConverter.cs`:

```csharp
using System.Globalization;
using System.Windows;
using System.Windows.Data;
using WeatherApp.Core;

namespace WeatherApp;

/// <summary>Visible when the bound Status equals the ConverterParameter status; else Collapsed.</summary>
public sealed class StatusToVisibilityConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
        => value is ViewModelStatus status && parameter is ViewModelStatus target && status == target
            ? Visibility.Visible : Visibility.Collapsed;

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
        => throw new NotSupportedException();
}
```

- [ ] **Step 4: Build**

Run: `dotnet build`
Expected: Build succeeded across all three projects.

- [ ] **Step 5: Tier-3 smoke (manual) — observe the real app**

Run: `dotnet run --project src/WeatherApp`
Expected:
- The window opens, briefly shows "Loading…", then shows **London**, a temperature like `24°C`, a Condition like `Overcast`, and `as of HH:mm`.
- A log file appears under `%LOCALAPPDATA%\SimpleDesktopWeatherApp\logs\log-*.txt` containing an INFO line with the Open-Meteo URL, `200`, and a latency (Seam 2 path observable).
- Disconnect the network and re-run: the window shows **"Couldn't load the weather."** and does **not** crash (Principle #3), and the log shows an ERROR line.

- [ ] **Step 6: Commit**

```bash
git add src/WeatherApp/MainWindow.xaml src/WeatherApp/MainWindow.xaml.cs src/WeatherApp/StatusToVisibilityConverter.cs
git commit -m "feat(app): MainWindow binds view-model; renders Loading/Loaded/Failed"
```

---

## Task 12: Full-suite green + final commit

- [ ] **Step 1: Run the whole PR-gate suite (Tier-1 only)**

Run: `dotnet test --filter "Tier!=Live"`
Expected: PASS — ConditionMapper, provider parse, query params, failure cases, logging, LocationOptions binding, view-model transitions. Live test excluded.

- [ ] **Step 2: Confirm a clean build with no warnings**

Run: `dotnet build -warnaserror`
Expected: Build succeeded, 0 warnings.

- [ ] **Step 3: Final commit if anything outstanding**

```bash
git status   # expect clean
```

---

## Definition of done (maps to spec)

- [ ] App launches, fetches London Current Conditions once, displays label · °C · Condition text · "as of HH:mm".
- [ ] A failed/slow fetch shows "Couldn't load the weather." and never crashes (Principle #3); fetch is async + cancellable with a bounded timeout (Principle #4).
- [ ] No secret stored anywhere (keyless Open-Meteo — Principle #1 not engaged).
- [ ] **Seam 1** covered by Task 4 (Tier-1 WireMock replay of captured fixture + malformed body) and Task 9 (Tier-2 live).
- [ ] **Seam 2** covered by Task 8 — a real Serilog file-sink Tier-1 test crossing the on-disk/host-OS channel (writes to a temp dir, asserts the file content + directory creation = (c-ii) (d)) plus a captured-logger check of INFO/ERROR (c-i); `%LOCALAPPDATA%`-specific path is the Task 11 Tier-3 observable.
- [ ] **Seam 3** covered by Task 6 (binding round-trip + fail-fast validation).
- [ ] Tier-1 runs every commit; Tier-2 is trait-gated off the PR gate; Tier-3 is the manual smoke in Task 11.

---

## Fix-pass log — feature-doc-gauntlet run 1 (2026-06-29)

Run 1 failed (`reason: feature-docs`) on a single root cause raised by `check-seam-cynicism`
(arriving as two findings). Resolution: option (a) — strengthen Seam 2's proof so it crosses its
declared channel (human-endorsed before editing).

| Root cause | Fix (Spec + Plan, jointly) | Closure check |
|---|---|---|
| Seam 2's automated (d) never crossed its declared `persistent-on-disk-state` + `host-OS` channel — the `CapturingLogger` mocked the sink; the on-disk contract was deferred to manual Tier-3. Its (c) also welded an in-process diagnostics contract to the on-disk contract. | **Spec Seam 2:** split (c) into **(c-i)** in-process diagnostics and **(c-ii)** on-disk/host-OS; rewrote (d) so (c-ii) is proven by a **real Serilog `File` sink → temp dir** Tier-1 test (asserts the file content on disk + directory creation), keeping the captured-logger check for (c-i). **Plan:** Task 1 adds `Serilog`/`Serilog.Sinks.File`/`Serilog.Extensions.Logging` to the Tests project; Task 8 gains the file-sink boundary-crossing test; coverage map + Definition of done updated. | `grep` for `mocks the sink` / `in-memory logger` as the seam's proof returns only the historical gauntlet sign-off (run-1 record), not any live contract restatement. Spec Seam 2 (d) now reads "real file I/O on the filesystem side"; Plan Task 8 writes through a live `File` sink and reads the bytes back. |

**Observations deliberately left (non-gating, off the edit path):**
- Add a `Condition` entry to `business-domain-context.md` — the glossary is higher-authority and
  domain-owned; a vocabulary addition belongs in `/grill-with-docs`, not a doc-fix pass. Left for the human.
- Seam 3's binding test uses a synthetic temp file rather than round-tripping the shipped
  `appsettings.json` — closing it would couple the Tests project to the WPF project's output
  (Tests references Core only by design). Not worth the coupling for a non-gating note; left as-is.

The conductor re-runs the **full** gauntlet (all three leaves, fresh) next — this fix pass clears nothing by itself.
