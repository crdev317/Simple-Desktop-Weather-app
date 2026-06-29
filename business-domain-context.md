# Domain Glossary

The language of **Simple Desktop Weather App**. Definitions only — no implementation, no process, no decisions. When a term refers to another defined term, that term is used by name to build the web.

## Active Location
The Saved Location whose weather is currently displayed. Exactly one Saved Location is the Active Location at any time. Distinct from the Default Location (which is merely the one shown at startup) — the Active Location changes as the user switches between Saved Locations.

## Current Conditions
The weather state at a Location *now* — the present Reading. Distinct from a Forecast, which describes future weather, and from a stale Last-Known Reading, which is a past Reading shown when a fresh one cannot be fetched.

## Daily Forecast
A Forecast expressed as one entry per day (e.g. high/low and a summary condition). Distinct from the Hourly Forecast, which is finer-grained.

## Default Location
The Saved Location shown as the Active Location when the app starts. One Saved Location is the Default Location. Not necessarily the first one added.

## Forecast
Predicted weather for a Location at future points in time. Distinct from Current Conditions (the present) and from a Reading (a single observed snapshot). A Forecast is composed of an Hourly Forecast and/or a Daily Forecast.

## Hourly Forecast
A Forecast expressed as one entry per hour. Distinct from the Daily Forecast, which is coarser-grained.

## Last-Known Reading
The most recent successfully-fetched Reading for a Location, retained so it can be shown when a fresh fetch fails. When displayed in place of a current Reading it is *stale* — the app marks it as such with the time it was taken. Not the same as Current Conditions, which is always a fresh present-time Reading.

## Location
A geographic place the app can show weather for, identified by name and/or coordinates. The general concept; a Location the user has added to their list is a Saved Location.

## Reading
A single snapshot of weather values for a Location at one point in time. Current Conditions is the present Reading; a Last-Known Reading is a retained past Reading. A Forecast is made of predicted future Readings rather than observed ones.

## Saved Location
A Location the user has added to their list and can switch to. The app holds zero or more Saved Locations; one is the Default Location, and one is the Active Location at any given moment. A plain Location becomes a Saved Location once added to the list.

## Stale
The state of a Reading that is no longer current — shown when a fresh fetch fails, always accompanied by the time the Reading was taken. A Last-Known Reading is what gets shown while Stale.
