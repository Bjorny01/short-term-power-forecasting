# ENTSO-E and entsoe-py — Guide for Power Data Extraction

---

## What is ENTSO-E and entsoe-py

ENTSO-E stands for the European Network of Transmission System Operators for Electricity. It is the organisation that brings together all 39 transmission system operators (TSOs) across Europe — Svenska Kraftnät for Sweden being one of them. As part of EU energy market regulation, all TSOs are required to submit their data to the ENTSO-E Transparency Platform, which acts as a centralised public database for European electricity data. This makes it the single most comprehensive source for cross-border comparable power data in Europe.

The Transparency Platform has a REST API, but it is based on XML and fairly verbose to work with directly. `entsoe-py` is an open source Python library that wraps that API and returns clean Pandas Series and DataFrames instead of raw XML. It was originally created by EnergieID, a Belgian energy data company, and is maintained on GitHub. It is the standard tool used in both academic and industry settings when working with ENTSO-E data in Python.

---

## What Data is Available

The data is organised around a few core categories.

**Load (consumption)**
- Actual load — measured total electricity consumption for a given area and time range
- Day-ahead load forecasts — what the TSOs predicted consumption would be the day before

**Generation**
- Actual aggregated generation broken down by production type — separate time series for wind onshore, wind offshore, solar, hydro, nuclear, and so on
- Day-ahead generation forecasts per production type — the TSO's predictions of what each generation type will produce the following day. These are real operational forecasts published by the TSO, not something you have to generate yourself

**Cross-border flows**
- Physical flows and scheduled exchanges between bidding areas — relevant for Sweden since it is heavily interconnected with Norway, Denmark, and Finland

**Sweden bidding areas**

| Code | Region |
|------|--------|
| SE_1 | North |
| SE_2 | North-central |
| SE_3 | South-central (includes Stockholm and Åland) |
| SE_4 | South (Malmö area) |

Every query is made at the bidding area level.

---

## How to Extract the Data

### Step 1 — Get an API key

Register at [transparency.entsoe.eu](https://transparency.entsoe.eu), after that contact them by email at transparency@entsoe.eu and ask for API key.

### Step 2 — Install the library

```bash
pip install entsoe-py
```

### Step 3 — Instantiate the client

The central object is `EntsoePandasClient`. You instantiate it once with your API key and then call methods on it.

```python
import pandas as pd
from entsoe import EntsoePandasClient

client = EntsoePandasClient(api_key="your-api-key-here")
```

### Step 4 — Define your time range

All methods take a `start` and `end` argument as `pd.Timestamp` objects with a timezone.

```python
start = pd.Timestamp("2023-01-01", tz="Europe/Stockholm")
end   = pd.Timestamp("2023-12-31", tz="Europe/Stockholm")
```

### Step 5 — Query data

**Actual load** — returns a Pandas Series indexed by timestamp at hourly resolution:

```python
load = client.query_load("SE_3", start=start, end=end)
```

**Actual generation by source** — returns a DataFrame where each column is a generation type (wind onshore, solar, hydro run-of-river, etc.):

```python
generation = client.query_generation("SE_3", start=start, end=end)
```

**Day-ahead wind and solar forecasts** — returns a DataFrame with separate columns for wind and solar forecast values, published the day before the delivery period:

```python
forecast = client.query_wind_and_solar_forecast("SE_3", start=start, end=end)
```

**Day-ahead load forecast:**

```python
load_forecast = client.query_load_forecast("SE_3", start=start, end=end)
```

---

## Practical Things to Be Aware Of

**Data availability** starts reliably from around 2015 for most data types, though some earlier data exists.

**Gaps** do occur — certain dates or hours may be missing depending on what the TSO submitted, so always check for and handle NaN values after querying.

**Rate limits** — for large historical pulls, query year by year in a loop rather than requesting several years in one call. The library raises a `NoMatchingDataError` if a query returns nothing, which you should catch when looping:

```python
from entsoe.exceptions import NoMatchingDataError

for year in range(2018, 2024):
    start = pd.Timestamp(f"{year}-01-01", tz="Europe/Stockholm")
    end   = pd.Timestamp(f"{year}-12-31", tz="Europe/Stockholm")
    try:
        load = client.query_load("SE_3", start=start, end=end)
        # save or process load
    except NoMatchingDataError:
        print(f"No data for {year}")
```

**Timezones** — timestamps are returned in UTC by default regardless of the timezone passed in for the query bounds. Be consistent about timezone conversion when merging with weather data from SMHI or Open-Meteo, which may come in local time.
