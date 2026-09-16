# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A [bullpen](https://github.com/MLB-LED-Scoreboard/mlb-led-scoreboard) plugin that displays Walt Disney World attraction wait times on an RGB LED matrix. It is installed alongside the MLB LED Scoreboard and registered via the `bullpen.mlbled.plugin` entry point.

## Setup

```bash
pip install -e .
```

The plugin depends on `lightninglane-live-led` (pulled from the `develop` branch of the sibling repo, per `pyproject.toml`). During local development the sibling repo lives at `../LightningLane-Live-LED`.

## Plugin architecture

The bullpen framework expects a single `load()` function (registered in `pyproject.toml` under `[project.entry-points.'bullpen.mlbled.plugin']`) that returns a `(Config, Data, Renderer)` triple. All three classes live in `src/lightninglane_plugin/__init__.py`.

| Class | Role |
|-------|------|
| `Config` | Reads plugin config keys (`parks`, `refresh_seconds`, `weather_api_key`, `trip_dates`) from `base.plugin_config` |
| `Data` | On first `update()` call, fetches the WDW park list then starts a daemon background thread (`live_data_updater`) that refreshes live wait-time data in-place on a shared list |
| `Renderer` | Phase-based cycle (Mickey intro → trip countdown → parks); within the parks phase, cycles through `data.open_parks()` (any operating park, even with zero displayable rides right now) and each park's displayable attractions one at a time (8 s per screen); calls `initialize_fonts` once on the first render pass |

## Dependency library layout (`lightninglane-live-led`)

The display and data logic lives in the dependency, not this plugin:

- `api/disney_api.py` — synchronous park/schedule fetching + async per-attraction live data via `aiohttp`; data source is the [ThemeParks Wiki API](https://api.themeparks.wiki)
- `updater/data_updater.py` — `live_data_updater()` loop; merges new live data into the shared `parks_data` list in-place so the renderer always sees the latest state
- `display/display.py` — font loading (`initialize_fonts`) keyed by matrix height (32 or 64); color definitions
- `display/park/` and `display/attractions/` — rendering functions called by `Renderer.render()`

## Running tests (in the dependency repo)

```bash
cd ../LightningLane-Live-LED
pytest
```

## Key config options (set in bullpen's `config.json`)

```json
{
  "plugin_config": {
    "parks": ["Magic Kingdom", "EPCOT"],   // omit to show all WDW parks
    "refresh_seconds": 300,
    "weather_api_key": "your-openweathermap-api-key",   // omit to skip the weather widget
    "trip_dates": ["2026-06-17"]
  }
}
```

## Weather

`display/park/park_details.py` (in the dependency) shows a weather widget on each park screen, sourced from OpenWeatherMap. The dependency's `fetch_weather_data()` normally reads its own `config.json`'s `weather.apikey`, but that file doesn't exist in the plugin's process, so there is currently no way to supply a key when running as a plugin — the weather widget is always omitted.

`Data.update()` does **not** pass `weather_api_key` to `live_data_updater()` — `develop`'s `live_data_updater()` signature has no such parameter, and passing it as a kwarg raises `TypeError` and crashes the whole background updater thread (not just weather), which silently breaks all live data with no error visible to the user. [PR #76](https://github.com/jc214809/LightningLane-Live-LED/pull/76) would add real support for this; until it merges, `Config.weather_api_key` is read from plugin config but intentionally unused.
