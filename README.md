# Solarcast

Solarcast is a SvelteKit app that estimates UK week-ahead solar generation using Open-Meteo forecast data and your system setup.  Uses browser localstorage and collects no data.
Demo: https://solarcast.alpower.com/

<img width="3470" height="2608" alt="CleanShot 2026-03-22 at 17 19 46@2x" src="https://github.com/user-attachments/assets/acda890b-4706-49a5-8166-9d8075ee8e0f" />


## Features

- UK location lookup by town or postcode
- Multiple panel strings, each with:
  - kWp
  - tilt
  - compass direction (azimuth)
  - loss percentage
  - calibration factor
- Battery and demand simulation:
  - battery capacity
  - initial SoC
  - round-trip efficiency
  - daily usage
- Export cap and curtailment modelling
- 7-day table with:
  - day/date
  - plain-English forecast
  - cloud %
  - sunny hours
  - generation/import/export/curtailed
  - end battery SoC
- Weekly chart with weather icons, cloud line, and grouped hover tooltip
- Persists settings and last forecast in localStorage

## Data Sources

- Open-Meteo Geocoding API
- Open-Meteo Forecast API:
  - `hourly=global_tilted_irradiance`
  - `hourly=temperature_2m,cloud_cover,direct_radiation`
  - `daily=weather_code`
- Postcodes.io for UK postcode coordinate lookup fallback

## Model Notes

- Uses hourly forecast timesteps.
- Export limit is an instantaneous power cap (`kW`), approximated per hourly timestep.
- Sunny hours are counted from hourly thresholds:
  - `direct_radiation >= sunnyDirectThresholdWm2`
  - `cloud_cover <= sunnyCloudMaxPercent`
- Temperature derating uses ambient temperature plus configurable module temperature rise.
- This is a practical planning model, not a physical-grade plant simulation.

## Develop

```sh
npm install
npm run dev -- --open
```

## Validate

```sh
npm run check
npm run build
```

## Deploy

This project currently uses `@sveltejs/adapter-auto`.

- For the easiest Git-based deployment, use Vercel.
- If needed, switch to a platform-specific adapter (for example Vercel/Netlify/Cloudflare) in `svelte.config.js`.
