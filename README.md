# Weather Data Processing Workflows

This repository contains GitHub Actions workflows that automatically fetch, process, and store weather data from NOAA's Global Forecast System (GFS) for use with MapLibre GL JS and WeatherLayers.

## Overview

Two workflows work together to provide comprehensive weather data coverage:

1. **Scheduled Weather Data** (`weather-data.yml`) - Runs automatically every 6 hours
2. **Historical Weather Data** (`pastdays-weather.yml`) - Manual trigger for backfilling data

## Data Sources

### NOAA Global Forecast System (GFS)
- **Source**: [NOAA NOMADS](https://nomads.ncep.noaa.gov)
- **Resolution**: 0.25° (~25km)
- **Update Frequency**: Every 6 hours (00Z, 06Z, 12Z, 18Z)
- **Data Availability**: Usually 3-4 hours after model run time
- **Forecast Hour**: f000 (analysis/initial conditions)

#### About f000 Data
The workflows specifically use **f000** forecast hour data.
This prefix in a GFS data file indicates the forecast hour. "f000" specifically denotes that this is the output 
from the very beginning of the model run—the "0-hour forecast". 
It essentially contains the initial conditions, or analysis, upon which the subsequent forecasts are based. 
This makes f000 data ideal for current-like "low resolution" global weather visualization, as opposed to forecast data (f003, f006, etc.) which would show predicted future conditions.

## Weather Variables

The workflows process five key meteorological variables:

| Variable | GRIB Parameter | Level | Description | Scale Range |
|----------|----------------|-------|-------------|-------------|
| **Wind U** | UGRD | 10m above ground | East-west wind component | -128 to 127 m/s → 0-255 |
| **Wind V** | VGRD | 10m above ground | North-south wind component | -128 to 127 m/s → 0-255 |
| **Temperature** | TMP | 2m above ground | Air temperature | -60 to 52°C → 0-255 |
| **Cloud Cover** | TCDC | Entire atmosphere | Total cloud coverage | 0-100% → 0-255 |
| **Reflectivity** | REFC | Entire atmosphere | Composite radar reflectivity | 0-60 dBZ → 0-255 |

## Output Format

### File Naming Convention
```
processed-data/[variable]_[YYYYMMDD]_[HH]z.png
```

Examples:
- `wind_20250823_12z.png`
- `temp_20250823_12z.png`
- `cloud_20250823_12z.png`
- `reflectivity_20250823_12z.png`

### WeatherLayers GL Format
Wind data is specially formatted for [WeatherLayers GL](https://weatherlayers.com) compatibility (compliant with [data sources specs](https://docs.weatherlayers.com/weatherlayers-gl/data-sources#example-wind-from-grib-to-png):
- **Red Channel**: U component (east-west wind)
- **Green Channel**: V component (north-south wind) 
- **Blue Channel**: V component (duplicated)

All other variables use single-channel grayscale PNG format.

## Workflows Details

### Scheduled Weather Data (`weather-data.yml`)

**Purpose**: Maintains current weather data automatically

**Schedule**: 
- Runs every 6 hours at 30 minutes past the hour
- Can be manually triggered via `workflow_dispatch`

**Logic**:
- Targets the latest GFS run whose data should be published (now − 4h, floored to the 6-hour grid), so it is robust to GitHub's cron delays
- Falls back to the previous run *as a whole* if any variable is unavailable — output files are always named after the run actually downloaded
- Skips processing when the target run is already in the repository
- Regenerates `processed-data/manifest.json` (per-variable list of available timestamps) so consumers can avoid the rate-limited GitHub API
- Processes and commits new data to the repository

**Data Retention**: Keeps last 3 days of data (configurable)

### Historical Weather Data (`pastdays-weather.yml`)

**Purpose**: Backfill missing historical data or batch process multiple days

**Trigger**: Manual only via GitHub Actions interface

**Options**:
- Choose 1-10 days of historical data to process
- Processes all 4 daily GFS runs (00Z, 06Z, 12Z, 18Z) for selected period
- Skips runs less than 5 hours old (data not yet available)

**Use Cases**:
- Initial repository setup
- Fill gaps from workflow failures
- Batch processing for analysis projects

## Processing Pipeline

### 1. Data Download
- Fetches GRIB2 files from NOAA NOMADS server
- Uses filtered downloads to get only required variables/levels
- Implements fallback logic for data availability issues

### 2. Format Conversion
- Converts GRIB2 to PNG using GDAL
- Applies appropriate scaling for each variable type
- Ensures 8-bit (Byte) output format for web compatibility

### 3. Quality Control
- Verifies all outputs are Byte format
- Lists downloaded files for debugging
- Handles missing data gracefully

### 4. Storage
- Commits processed PNGs to repository
- Uses descriptive commit messages with timestamps
- Automatically pushes to main branch

### GDAL Tools Used
- `gdalinfo` - Inspect GRIB metadata
- `gdalbuildvrt` - Create virtual datasets for multi-band wind data  
- `gdal_translate` - Convert formats and apply scaling

## Usage with MapLibre GL JS

For WeatherLayers GL integration, please check the official documentation at https://docs.weatherlayers.com/weatherlayers-gl
