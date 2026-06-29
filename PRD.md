# PRD — Simple Desktop Weather App

> Product requirements for a Windows desktop weather app built on .NET 10 / WPF (MVVM), using Open-Meteo as the weather and geocoding provider. Vocabulary follows `business-domain-context.md`; engineering constraints follow `Technical-Context.MD`.

## Problem Statement

A user on Windows wants to glance at the weather for the places they care about without opening a browser, signing into a website, or wading through ads. They have more than one place of interest (home, work, family, a trip), and they want both what it's like *now* and what's coming. When their connection drops or the weather service is briefly unavailable, a blank or crashing app is worse than slightly old data — they still want to see the last thing the app knew, clearly marked as not current.

## Solution

A lightweight desktop app that shows the **Current Conditions** and a **Forecast** (hourly and daily) for the user's **Active Location**, with a list of **Saved Locations** they can switch between and a **Default Location** shown at startup. The user adds places by **Place Search** (typing a name and picking from the **Place Results**, which disambiguates duplicates like the two Londons). Measurements display according to the user's **Unit Preference** (metric/imperial). When a refresh fails, the app shows the **Last-Known Reading** marked **Stale** with the time it was taken, and never crashes on a failed or slow fetch.

## Requirements

1. As a user, I want to see the Current Conditions for my Active Location, so that I know the weather right now without leaving my desktop.
2. As a user, I want to see an Hourly Forecast for my Active Location, so that I can plan the next few hours.
3. As a user, I want to see a Daily Forecast for my Active Location, so that I can plan the coming days.
4. As a user, I want each weather value labelled with what it is (temperature, wind, precipitation, etc.), so that I can read it at a glance.
5. As a user, I want to add a place by typing its name, so that I don't need to know its coordinates.
6. As a user, when my typed name matches several places, I want to see candidate Place Results with enough context (region/country) to tell them apart, so that I pick the right one.
7. As a user, I want to select one Place Result to add it as a Saved Location, so that I can return to it later.
8. As a user, I want to maintain a list of Saved Locations, so that I can track the weather for all the places I care about.
9. As a user, I want to switch the Active Location between my Saved Locations, so that I can view weather for any of them.
10. As a user, I want to remove a Saved Location, so that my list stays relevant.
11. As a user, I want to designate one Saved Location as the Default Location, so that it is shown when I open the app.
12. As a user, when I remove the Default Location, I want the app to keep working with another Saved Location as default, so that startup is never broken.
13. As a user, when I have no Saved Locations yet, I want a clear empty/onboarding state inviting me to add one via Place Search, so that I know what to do first.
14. As a user, I want my Saved Locations, my Default Location, and my Unit Preference to persist between sessions, so that I don't reconfigure the app each time.
15. As a user, I want to choose a Unit Preference (metric or imperial), so that values are shown in units I understand.
16. As a user, I want my Unit Preference applied consistently across Current Conditions and the Forecast for every Location, so that the display is coherent.
17. As a user, I want the app to refresh the weather for the Active Location, so that I see up-to-date information.
18. As a user, when a refresh fails, I want to keep seeing the Last-Known Reading marked Stale with the time it was taken, so that I still have useful information and know it isn't current.
19. As a user, I want the app to never crash or freeze when a fetch is slow or fails, so that it stays usable on a flaky connection.
20. As a user, I want the app to remain responsive while weather is being fetched, so that the window never locks up.
21. As a user, I want a clear, plain-language message when something goes wrong (no error codes, stack traces, or raw API output), so that I'm not confused by technical detail.
22. As a user, I want the displayed weather Condition shown as readable text (and/or an icon), so that I don't have to interpret raw codes.
23. As a user, I want the app to work on the Windows versions it supports, so that it runs on my machine.
24. As a user, I want the app to keep my data on my machine (no telemetry leaving the device), so that my location choices stay private.

## Implementation Decisions

**Stack & architecture.** .NET 10 / WPF with the MVVM pattern (per `Technical-Context.MD`). View-models hold no UI-thread dependencies so they are unit-testable. Composition via `Microsoft.Extensions.DependencyInjection`/`Hosting`; HTTP via `IHttpClientFactory`; JSON via `System.Text.Json`.

**Provider.** Open-Meteo is the single external provider, used for both weather and geocoding. It is keyless for non-commercial use, so no secret is stored today — but Overriding Principle #1 (secrets via Windows Credential Manager) applies the moment any keyed service is added.

**Deep modules** (small, stable interfaces; the HTTP clients and stores are the deep ones, the mappers are pure helpers, the view-models are the domain↔UI seam):

- **Weather Provider** — fetches Current Conditions and a Forecast (Hourly + Daily) for a Location. Interface along the lines of `GetCurrentConditions(coords, units, ct)` and `GetForecast(coords, units, ct)`, returning domain Readings / Forecast Entries. Encapsulates HTTP, JSON parsing, and unit-parameter wiring. All calls are async and accept a `CancellationToken` (Principle #4).
- **Place Search** — resolves a typed name to zero or more Place Results via Open-Meteo geocoding: `Search(name, ct)`. A Place Result is a coordinate pair + canonical label + region/country context.
- **Saved Locations Store** — persists the list of Saved Locations, the Default Location designation, and the Unit Preference, locally under `%LOCALAPPDATA%`. Interface around `List / Add / Remove / SetDefault / GetPreference / SetPreference`. Enforces coordinate-identity (no duplicate place under a different label) and the invariant that Default is absent only when the list is empty.
- **Last-Known Store** — retains the latest successful Reading per Location so it can be shown Stale on a failed refresh: `Save(reading)`, `GetLastKnown(loc)`.
- **Condition Mapper** — pure function mapping Open-Meteo WMO weather codes to a display Condition (text and/or icon key).
- **Unit Formatter** — pure function formatting a weather value for display according to the Unit Preference; presentation only, never alters observed/predicted values.
- **View-Models** — orchestrate fetch for the Active Location, expose observable display state, drive the Stale fallback (show Last-Known Reading + time when a refresh fails), and keep all I/O off the UI thread.

**Identity & data.** Location identity is coordinates; names are display labels (per the glossary). Persistence is local JSON files; no database, no cloud sync. No telemetry leaves the machine; Serilog logs to a rolling file under `%LOCALAPPDATA%` (every Open-Meteo request with status+latency, and every failure).

**Degraded behaviour.** A failed or slow refresh degrades to the Last-Known Reading marked Stale; the UI surfaces a plain-language message and never throws to the user (Principle #3). Staleness is failure-driven, not age-based.

## Testing Decisions

**What makes a good test here:** assert on the deterministic envelope — parsed domain shape (Readings, Forecast Entries, Place Results), view-model state transitions, the Stale-fallback behaviour, and file-persistence round-trips — never on the exact wording of Open-Meteo's payload. Test the contract, not the provider. Every seam gets a real-IO test on at least one side (mock-on-both-sides only proves the mocks agree).

**Modules to be tested (all of them, per decision):**

- **HTTP clients (Weather Provider + Place Search)** — Tier 1 recorded-replay against the Open-Meteo HTTP seam via `WireMock.Net` (replayed response fixtures), plus Tier 2 live calls against real Open-Meteo (bounded/scheduled). These are the project's primary external seam.
- **Stores (Saved Locations + Last-Known)** — real local file I/O (Tier 1): persistence round-trips, coordinate-identity dedup, Default-designation invariants, empty-list edges.
- **Pure helpers (Condition Mapper + Unit Formatter)** — table-driven unit tests over the WMO code range and each unit system. Cheap and high-value.
- **View-Models** — orchestration and Stale-fallback logic tested off the UI thread, using fake stores and a faked Weather Provider seam, asserting on observable state.

**Tiers & the ratchet** (per `Technical-Context.MD` → Testing & the ratchet): three tiers (recorded-replay every commit; live-disposable scheduled with a cost ceiling; live continuous/manual smoke); the platform matrix runs Tier 1 + Tier 2 on every supported Windows version for the file-I/O-touching code; every Tier-3 defect pins a cheap-tier regression bounded by the seam where it surfaced. Coverage is planned at the Feature level on the Roadmap, not per Story.

**Prior art:** none yet — this is a greenfield repo, so these tests establish the prior art for later Features.

## Out of Scope

- **Severe-weather alerts / toast notifications** — explicitly deferred (User Feedback decision: in-app only for now).
- **Multiple weather providers / provider failover** — Open-Meteo only.
- **Weather maps, radar, satellite imagery, historical/archive weather.**
- **Cloud sync, accounts, or sharing** — all data stays local.
- **Cross-platform support** — Windows only (WPF).
- **Localization/translation of UI copy** beyond what's trivially available.
- **Age-based staleness thresholds** — staleness is failure-driven (decided in the glossary).
- **Auto-detected current location via OS geolocation** — Locations are added by Place Search (could be revisited later).
- **Per-quantity unit overrides** — a single Unit Preference (metric/imperial) for now.

## Further Notes

- This product is multi-feature; the next step is `/roadmap` to break these Requirements into a sequenced Feature list, each becoming a `/brainstorming` → Spec → Plan.
- The Requirements above are product-level requirements that feed the Roadmap — not the orchestrator's atomic User Story work items (those are created later by `/enate-to-stories`).
- Open questions worth resolving during roadmapping: the refresh trigger (manual vs interval) — currently named only as a requirement, deliberately kept out of the glossary as process; and whether the Condition display ships with icons in v1 or text only.
