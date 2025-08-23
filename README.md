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
- - **Forecast Hour**: f000 (analysis/current conditions)

#### About f000 Data
The workflows specifically use **f000** forecast hour data, which represents:
- **Analysis Data**: Not a forecast, but the model's best estimate of current atmospheric conditions
- **Data Assimilation**: Combines observations from weather stations, satellites, radiosondes, and aircraft
- **Real-time Conditions**: Represents the atmospheric state at the model run time (00Z, 06Z, 12Z, 18Z)
- **Highest Accuracy**: Most accurate data available, as it's based on actual observations rather than predictions
- **Immediate Availability**: Available as soon as the model analysis cycle completes (~3-4 hours after run time)

This makes f000 data ideal for current weather visualization and near real-time applications, as opposed to forecast data (f003, f006, etc.) which would show predicted future conditions.

## Weather Variables

The workflows process five key meteorological variables:

| Variable | GRIB Parameter | Level | Description | Scale Range |
|----------|----------------|-------|-------------|-------------|
| **Wind U** | UGRD | 10m above ground | East-west wind component | -127 to 128 m/s → 0-255 |
| **Wind V** | VGRD | 10m above ground | North-south wind component | -127 to 128 m/s → 0-255 |
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
- Determines the latest available GFS run based on current time
- Accounts for 3-4 hour data availability delay
- Falls back to previous runs if current data is unavailable
- Processes and commits new data to the repository

**Data Retention**: Keeps last 20 days of data (configurable)

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

## Dependencies

### System Requirements
- Ubuntu Linux (GitHub Actions runner)
- GDAL/OGR with Python bindings
- Standard Unix tools (curl, jq, date, find)

### GDAL Tools Used
- `gdalinfo` - Inspect GRIB metadata
- `gdalbuildvrt` - Create virtual datasets for multi-band wind data  
- `gdal_translate` - Convert formats and apply scaling

## Usage with MapLibre GL JS

For WeatherLayers GL integration, please check the official documentation at https://docs.weatherlayers.com/weatherlayers-gl

## Monitoring and Troubleshooting

### Common Issues
- **Data availability delays**: GFS runs may be delayed; workflows include fallback logic
- **Network timeouts**: NOAA servers occasionally timeout; retry logic handles this
- **Disk space**: Old data is automatically cleaned up (20-day retention)

### Logs
- Each workflow step provides detailed logging
- Failed downloads are logged but don't stop the workflow
- Successful processing includes file verification output

### Manual Intervention
If workflows fail:
1. Check NOAA server status
2. Run Historical Weather Data workflow to backfill
3. Review GitHub Actions logs for specific error messages
