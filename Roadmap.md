# Roadmap

**Product:** Simple Desktop Weather App — a Windows desktop app (.NET 10 / WPF) showing current conditions and forecasts for the user's saved locations, powered by Open-Meteo.
**Last reviewed:** 2026-06-29

## Sequencing

Features are listed in delivery order. Each Feature gets its own `/brainstorming` session, Spec, and Plan. Vocabulary follows `business-domain-context.md`; engineering constraints follow `Technical-Context.MD`.

---

## Feature 1: Show Current Conditions for one location 🔫 *tracer bullet*

**Status:** **Published to ADO** — Feature [#95069](https://dev.azure.com/Enate/Factory%20DevTest/_workitems/edit/95069), 2026-06-29. Gauntlet: pass.

On launch, the app fetches and displays the Current Conditions for a single hard-coded Location (a fixed coordinate) from Open-Meteo, showing temperature and a readable Condition in a WPF window.

**Out of scope:** Place Search, the Saved Locations list, Default/Active switching, Forecast, Unit Preference (fixed to metric), and the full Stale/Last-Known fallback — a basic "couldn't load" message is enough, but the app must not crash.

**Dependencies:** None (this is the tracer bullet).

**Why first:** it pierces every layer end-to-end — WPF view ↔ view-model ↔ Weather Provider HTTP client ↔ Open-Meteo ↔ JSON→Reading ↔ Condition Mapper ↔ Serilog — proving the whole stack and the testing harness (WireMock replay + a live call) work, with nothing else in the way.

---

## Feature 2: Place Search — choose the location

The user types a place name, the app runs a Place Search against Open-Meteo's geocoding API and shows the candidate Place Results (with region/country context to tell duplicates like the two Londons apart). The user picks one, and the app shows its Current Conditions, replacing Feature 1's hard-coded Location. The chosen Location is the single Active Location for the session.

**Out of scope:** persisting the choice across restarts (Feature 3), a list of multiple locations, Default Location, Forecast, and Unit Preference.

**Dependencies:** Feature 1 (reuses the Weather Provider client, the Reading→display path, and the Condition Mapper). Adds the Place Search client (second Open-Meteo HTTP seam) and a search/pick UI.

---

## Feature 3: Saved Locations — a persistent list you switch between

The user builds a list of Saved Locations (added via Feature 2's Place Search), switches the Active Location between them, removes ones they don't want, and designates one as the Default Location shown at startup. The list, the Default designation, and the Active selection persist across restarts. With an empty list, the app shows the empty/onboarding state inviting a Place Search.

**Out of scope:** Forecast, Unit Preference, and the Stale/Last-Known fallback. Coordinate-identity dedup *is* in scope (the same place cannot be saved twice).

**Dependencies:** Feature 2 (Place Search is how a Location is added) and Feature 1 (the per-Location Current Conditions display). Introduces the Saved Locations Store (first real file-I/O seam, under `%LOCALAPPDATA%`), enforcing coordinate-identity and the invariant that the Default Location is absent only when the list is empty.

---

## Feature 4: Forecast — hourly and daily

For the Active Location, the app shows a Forecast alongside the Current Conditions — an Hourly Forecast (per-hour Forecast Entries) and a Daily Forecast (per-day entries with high/low and a summary Condition). Forecast Entries reuse the Condition Mapper.

**Out of scope:** Unit Preference (forecast shown in fixed metric until Feature 5) and the Stale/Last-Known fallback.

**Dependencies:** Feature 1 (extends the Weather Provider with a forecast call on the same Open-Meteo HTTP seam, and reuses the Condition Mapper) and Feature 3 (forecast is shown for the current Active Location). Adds the forecast display.

---

## Feature 5: Unit Preference — metric / imperial

The user chooses a Unit Preference (metric or imperial), applied consistently to display — Current Conditions and Forecast Entries, across every Location. The preference persists across restarts. Presentation only: it changes how values are shown, never the observed or predicted data.

**Out of scope:** per-quantity unit overrides (a single metric/imperial choice) and the Stale fallback.

**Dependencies:** Feature 3 (the Saved Locations Store is extended to persist the preference) and Feature 4 (so the toggle reformats Forecast as well as Current Conditions). Introduces the Unit Formatter (pure helper) and the preference UI control.

---

## Feature 6: Resilient refresh & the Stale fallback

The app refreshes the Active Location's weather, and when a refresh fails it shows the Last-Known Reading marked Stale with the time it was taken — never crashing or freezing on a slow or failed fetch, and never showing error codes or stack traces (plain language only). This fully delivers Overriding Principle #3, replacing Feature 1's basic "couldn't load" message.

**Out of scope:** age-based staleness (staleness stays failure-driven) and any multi-provider failover.

**Dependencies:** Feature 1 (the Reading fetch path) and Feature 3 (per-Location persistence). Introduces the Last-Known Store (file-I/O seam retaining the latest successful Reading per Location) and the Stale display state. The refresh trigger (manual vs interval) is settled in this Feature's `/brainstorming` session.
