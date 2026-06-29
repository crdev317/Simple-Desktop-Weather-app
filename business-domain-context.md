# Domain Glossary

The language of **Simple Desktop Weather App**. Definitions only — no implementation, no process, no decisions. When a term refers to another defined term, that term is used by name to build the web.

## Active Location
The Saved Location whose weather is currently displayed. Exactly one Saved Location is the Active Location **when the list is non-empty**; with no Saved Locations there is no Active Location (the app shows an empty/onboarding state). Distinct from the Default Location (which is merely the one shown at startup) — the Active Location changes as the user switches between Saved Locations.

## Current Conditions
The weather state at a Location *now* — the present Reading. Distinct from a Forecast, which describes future weather, and from a stale Last-Known Reading, which is a past Reading shown when a fresh one cannot be fetched.

## Daily Forecast
A Forecast expressed as one Forecast Entry per day (e.g. high/low and a summary condition). Distinct from the Hourly Forecast, which is finer-grained.

## Default Location
The single user-designated Saved Location shown as the Active Location when the app starts. Exactly one Saved Location is the Default Location **when the list is non-empty**; with no Saved Locations there is no Default Location. If the Default Location is removed, the designation moves to another Saved Location (or becomes absent when the list empties). Not necessarily the first one added.

## Forecast
Predicted weather for a Location at future points in time, composed of Forecast Entries. Distinct from Current Conditions (the present) and from Readings (which are observed, not predicted). A Forecast is organised as an Hourly Forecast and/or a Daily Forecast.

## Forecast Entry
A single **predicted** weather value for a Location at one future point in time. The predicted counterpart of a Reading (which is observed). An Hourly Forecast is a sequence of per-hour Forecast Entries; a Daily Forecast is a sequence of per-day Forecast Entries.

## Hourly Forecast
A Forecast expressed as one Forecast Entry per hour. Distinct from the Daily Forecast, which is coarser-grained.

## Last-Known Reading
The most recent successfully-fetched Reading for a Location, retained so it can be shown when a fresh fetch fails. When displayed in place of a current Reading it is *stale* — the app marks it as such with the time it was taken. Not the same as Current Conditions, which is always a fresh present-time Reading.

## Location
A geographic place the app can show weather for. Its **identity is its coordinates** (latitude/longitude); its name is a human-friendly display label, not identity. Two Locations are the same place when their coordinates match, regardless of label. The general concept; a Location the user has added to their list is a Saved Location.

## Place Result
A candidate Location returned by a Place Search — a resolved coordinate pair with a canonical label (and usually region/country context to tell duplicates apart). Not yet a Saved Location: the user selects one Place Result to add to their list. Several Place Results may share a name (the two Londons) but differ in coordinates.

## Place Search
The activity of looking up candidate Locations by a typed name, returning zero or more Place Results for the user to choose from. It is how a name becomes a concrete Location; distinct from selecting an already-Saved Location.

## Reading
A single snapshot of **observed** weather values for a Location at one point in time — what the weather actually is or was. Current Conditions is the present Reading; a Last-Known Reading is a retained past Reading. A Reading is never a prediction: a predicted future value is a Forecast Entry, not a Reading.

## Saved Location
A Location the user has added to their list and can switch to. The app holds zero or more Saved Locations; one is the Default Location, and one is the Active Location at any given moment. A plain Location becomes a Saved Location once added to the list. Because identity is coordinates, the same place cannot be saved twice even if added under different labels.

## Stale
The state of a displayed Reading when the app could not refresh it — i.e. a refresh was attempted and failed. Staleness is driven by **fetch failure, not by age**: a successfully-fetched Reading is treated as current however old it is, and the displayed time lets the user judge its age. A Last-Known Reading is what gets shown while Stale, always accompanied by the time it was taken.

## Unit Preference
The user's chosen measurement system for *displaying* weather values — e.g. metric vs imperial (°C/°F, km/h vs mph, mm vs in). It governs presentation only: it changes how a Reading or Forecast Entry is shown, never what was observed or predicted. A single preference applies across all Locations.
