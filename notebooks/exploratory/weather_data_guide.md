# Weather Data Sources — SMHI and Open-Meteo

---

## Part 1 — SMHI Meteorological Observations

### What is SMHI

SMHI (Sveriges Meteorologiska och Hydrologiska Institut) is the Swedish government agency responsible for weather observation, forecasting, and climate research. Its observation network spans hundreds of physical stations across Sweden, collecting temperature, wind speed, wind direction, solar radiation, pressure, humidity, and precipitation. All of this observational data is made publicly available under a Creative Commons Attribution 4.0 licence at no cost and with no API key required.

The data is served through a REST API at `opendata-download-metobs.smhi.se`. The structure is hierarchical: you first select a **parameter** (the variable you want, e.g. wind speed), then a **station** (the physical measurement location), and finally a **time period** (latest months or the full corrected archive).

It is worth noting that SMHI distinguishes between two station types: **CORE** stations, which are actively monitored, inspected, and quality-controlled by SMHI, and **ADDITIONAL** stations, whose values are not quality checked. For ML training data you should prefer CORE stations.

---

### Key parameters relevant to power forecasting

| Parameter ID | Description | Unit |
|---|---|---|
| 4 | Wind speed (10-minute mean) | m/s |
| 3 | Wind direction | degrees |
| 21 | Wind gust | m/s |
| 1 | Temperature (instantaneous) | °C |
| 5 | Precipitation (past hour) | mm |
| 11 | Global irradiance | W/m² |
| 23 | Relative humidity | % |

The full parameter list is available at `opendata.smhi.se/apidocs/metobs/parameter.html`.

---

### Python approach — direct REST API with requests

There is a community library called `smhi-open-data` on PyPI, but it is lightly maintained. For anything beyond exploratory use, it is more reliable to call the REST API directly with `requests`, which gives you full control.

**Install:**

```bash
pip install requests pandas
```

**Step 1 — Find the closest station to your coordinates:**

```python
import requests
import pandas as pd
import math

def haversine(lat1, lon1, lat2, lon2):
    """Calculate distance in km between two lat/lon points."""
    R = 6371
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = math.sin(dlat/2)**2 + math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) * math.sin(dlon/2)**2
    return R * 2 * math.asin(math.sqrt(a))

# Parameter 4 = wind speed — get all stations measuring this parameter
PARAMETER_ID = 4
url = f"https://opendata-download-metobs.smhi.se/api/version/1.0/parameter/{PARAMETER_ID}.json"
response = requests.get(url)
stations = response.json()["station"]

# Your target location — e.g. Stockholm
target_lat, target_lon = 59.33, 18.07

# Find closest active station
closest = min(
    [s for s in stations if s.get("active", False)],
    key=lambda s: haversine(target_lat, target_lon, s["latitude"], s["longitude"])
)

print(f"Closest station: {closest['name']} (ID: {closest['id']})")
print(f"Distance: {haversine(target_lat, target_lon, closest['latitude'], closest['longitude']):.1f} km")
```

**Step 2 — Download the full historical archive for that station:**

```python
station_id = closest["id"]

# "corrected-archive" returns the full quality-controlled historical record as CSV
data_url = (
    f"https://opendata-download-metobs.smhi.se/api/version/1.0/"
    f"parameter/{PARAMETER_ID}/station/{station_id}/period/corrected-archive/data.csv"
)

# The CSV has a multi-line header — skip it by finding where data begins
raw = requests.get(data_url).text
lines = raw.split("\n")

# Find the header row (contains "Datum" or "Date")
header_idx = next(i for i, line in enumerate(lines) if "Datum" in line or "Date" in line)

from io import StringIO
df = pd.read_csv(
    StringIO("\n".join(lines[header_idx:])),
    sep=";",
    parse_dates=[[0, 1]],   # combine date and time columns
    dayfirst=True
)

df.columns = ["datetime"] + list(df.columns[1:])
df = df.set_index("datetime")
print(df.head())
```

The `corrected-archive` period gives you the full cleaned historical record. For recent data not yet in the archive, use `latest-months` instead.

---

### SMHI MESAN — gridded analysis (alternative to station data)

SMHI also provides **MESAN**, a gridded meteorological analysis product that interpolates observations onto a regular grid covering Sweden at roughly 2.5 km resolution. This can be more convenient than working with individual station data because you get consistent spatial coverage rather than point measurements from sparse stations. It is served through a separate endpoint at `opendata-download-metanalys.smhi.se`.

---

---

## Part 2 — Open-Meteo Historical Forecast API

### What is Open-Meteo

Open-Meteo is an open-source weather API project that aggregates numerical weather prediction (NWP) model output from multiple national weather services — including ECMWF, NOAA, DWD (Germany), and MET Norway — and exposes it through a free, no-registration JSON API. For Sweden the relevant high-resolution model is **ECMWF IFS** and **MET Norway** (the Nordic region model), with a spatial resolution of roughly 1–11 km depending on the model.

There are two distinct APIs relevant to ML training, and understanding the difference is important.

---

### Historical Weather API vs Historical Forecast API

**Historical Weather API** (`/v1/archive`) is based on reanalysis data, primarily ERA5. It offers data from 1940 onwards with high consistency across the entire time series, making it well suited for long-term climate analysis and trend studies. However, reanalysis is produced retrospectively by re-running a weather model over past data — it is not the same data structure that a live forecasting system produces.

**Historical Forecast API** is the more important one for your use case. Rather than reanalysis, it archives the actual output of live NWP forecast model runs. Each update to the weather model is saved, and the first few hours of each model run are stitched together into a continuous time series. This means the data has the same structure, biases, and characteristics as the forecast input your trained model will consume at inference time — which is a significant advantage when training ML models for operational forecasting.

---

### Python approach — openmeteo-requests library

Open-Meteo provides an official Python library that uses FlatBuffers instead of JSON for efficient data transfer, which matters when pulling large historical datasets.

**Install:**

```bash
pip install openmeteo-requests requests-cache retry-requests
```

**Step 1 — Set up the client with caching:**

```python
import openmeteo_requests
import requests_cache
import pandas as pd
from retry_requests import retry

# Cache responses locally to avoid re-downloading the same data
cache_session = requests_cache.CachedSession(".cache", expire_after=-1)  # -1 = cache forever
retry_session = retry(cache_session, retries=5, backoff_factor=0.2)
openmeteo = openmeteo_requests.Client(session=retry_session)
```

**Step 2 — Query the Historical Forecast API:**

The endpoint for historical forecast data is `/v1/forecast` with a `start_date` and `end_date` in the past.

```python
# Coordinates for Stockholm — replace with your target location
url = "https://historical-forecast-api.open-meteo.com/v1/forecast"

params = {
    "latitude": 59.33,
    "longitude": 18.07,
    "start_date": "2020-01-01",
    "end_date": "2023-12-31",
    "hourly": [
        "temperature_2m",           # 2-metre air temperature (°C)
        "wind_speed_10m",           # 10-metre wind speed (km/h)
        "wind_direction_10m",       # 10-metre wind direction (degrees)
        "wind_speed_100m",          # 100-metre wind speed — relevant for wind turbines
        "wind_direction_100m",
        "shortwave_radiation",      # Global horizontal irradiance (W/m²)
        "direct_normal_irradiance", # DNI — relevant for solar tracking systems
        "diffuse_radiation",        # Diffuse horizontal irradiance
    ],
    "models": "best_match",        # Let Open-Meteo select the best model for the location, metno_seamless is the MET Norway model
    "timezone": "Europe/Stockholm"
}

responses = openmeteo.weather_api(url, params=params)
response = responses[0]
```

**Step 3 — Convert to a Pandas DataFrame:**

```python
hourly = response.Hourly()

# Variables are returned in the same order as requested in params["hourly"]
df = pd.DataFrame({
    "datetime":                  pd.date_range(
                                     start=pd.to_datetime(hourly.Time(), unit="s", utc=True),
                                     end=pd.to_datetime(hourly.TimeEnd(), unit="s", utc=True),
                                     freq=pd.Timedelta(seconds=hourly.Interval()),
                                     inclusive="left"
                                 ),
    "temperature_2m":            hourly.Variables(0).ValuesAsNumpy(),
    "wind_speed_10m":            hourly.Variables(1).ValuesAsNumpy(),
    "wind_direction_10m":        hourly.Variables(2).ValuesAsNumpy(),
    "wind_speed_100m":           hourly.Variables(3).ValuesAsNumpy(),
    "wind_direction_100m":       hourly.Variables(4).ValuesAsNumpy(),
    "shortwave_radiation":       hourly.Variables(5).ValuesAsNumpy(),
    "direct_normal_irradiance":  hourly.Variables(6).ValuesAsNumpy(),
    "diffuse_radiation":         hourly.Variables(7).ValuesAsNumpy(),
})

df = df.set_index("datetime")
print(df.head())
```

Note that the variable index in `hourly.Variables(i)` must match the order of variables in the `params["hourly"]` list exactly.

---

### Key variables for power forecasting

| Variable name | Description | Relevance |
|---|---|---|
| `temperature_2m` | Air temperature at 2 m | Demand forecasting (heating/cooling loads) |
| `wind_speed_10m` | Wind speed at 10 m | Standard meteorological height |
| `wind_speed_100m` | Wind speed at 100 m | Closer to typical wind turbine hub height |
| `wind_direction_10m` / `100m` | Wind direction | Can affect ramp forecasting |
| `shortwave_radiation` | Global horizontal irradiance | Total solar input — key for solar generation |
| `direct_normal_irradiance` | Direct normal irradiance | Tracking solar installations |
| `diffuse_radiation` | Diffuse irradiance | Overcast / indirect solar contribution |

---

### Practical things to be aware of

**No API key required** for non-commercial use. If you are using this for research or a commercial product, check Open-Meteo's licensing terms.

**Caching is important** for large date ranges. The `requests-cache` integration shown above stores responses locally in a `.cache.sqlite` file so repeated runs during development do not re-download the same data.

**`best_match` vs explicit model** — the `best_match` option lets Open-Meteo select the most appropriate high-resolution model for your coordinates, which is usually ECMWF IFS or a regional model for Sweden. If you want to explicitly use a specific model (e.g. for consistency across a long training run), you can specify `"models": "ecmwf_ifs025"` instead.

**Timezone handling** — pass `"timezone": "Europe/Stockholm"` to get timestamps in local time, or omit it to receive UTC. Be consistent with whatever you use for your power data from ENTSO-E.

**Historical Forecast API data availability** starts from roughly 2017 onwards for most models. For earlier historical data, switch to the Historical Weather API (`/v1/archive`) which covers back to 1940 using ERA5 reanalysis.
