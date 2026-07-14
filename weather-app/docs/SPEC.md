# SkyLog — Product Specification

## What is SkyLog

SkyLog is a lightweight API that tracks weather conditions for saved locations. It accepts a city name (or a list of cities), resolves coordinates via geocoding, fetches current weather from the Open-Meteo API, and returns a structured weather report.

It stores every location and its weather snapshot locally so you can build a dashboard of conditions across multiple cities.

## API Contract

### POST /locations

Add a location and fetch its current weather.

**Request:**
```json
{
  "city": "San Francisco",
  "country_code": "US"
}
```

`city` is required. `country_code` is optional — an ISO 3166-1 alpha-2 code (e.g. `US`, `GB`, `JP`) used to disambiguate cities with the same name. If omitted, the geocoding API returns its best match.

**Success response (200):**
```json
{
  "id": 1,
  "city": "San Francisco",
  "country": "United States",
  "country_code": "US",
  "latitude": 37.7749,
  "longitude": -122.4194,
  "temperature": 11.3,
  "humidity": 88,
  "wind_speed": 10.9,
  "precipitation": 0.0,
  "weather_code": 3,
  "condition": "cloudy",
  "fetched_at": "2026-04-07T14:30:00Z"
}
```

**Error response when Open-Meteo API is unreachable (503):**
```json
{
  "detail": "Weather service is temporarily unavailable. Try again later."
}
```

**Error response when city cannot be geocoded (404):**
```json
{
  "detail": "City not found. Check the spelling or add a country_code to narrow the search."
}
```

**Error response for invalid input (422):**
Standard FastAPI validation error format.

---

### POST /locations/import

Add multiple locations from a plain-text list.

**Request:**
```json
{
  "content": "San Francisco, US\nLondon\n# skip this comment\nTokyo, JP"
}
```

Parsing rules for `content`:
- One location per line.
- Skip empty lines and lines starting with `#`.
- Accept the format `city` or `city, country_code`. Whitespace around the comma is trimmed.
- Lines where the city cannot be geocoded are collected in a `skipped` array in the response.
- Duplicate cities (same name and country_code as an existing location) are skipped with a note.

**Success response (200):**
```json
{
  "total_lines": 4,
  "added": 3,
  "skipped": [
    {
      "line": "# skip this comment",
      "reason": "comment"
    }
  ],
  "results": [
    {
      "city": "San Francisco",
      "country_code": "US",
      "condition": "cloudy",
      "temperature": 11.3
    },
    {
      "city": "London",
      "country_code": "GB",
      "condition": "rain",
      "temperature": 8.1
    },
    {
      "city": "Tokyo",
      "country_code": "JP",
      "condition": "clear",
      "temperature": 22.5
    }
  ],
  "fetched_at": "2026-04-07T14:30:00Z"
}
```

---

### GET /locations

Return saved locations with their latest weather data.

**Query parameters:**
| Param     | Type   | Default | Description                        |
|-----------|--------|---------|------------------------------------|
| `limit`   | int    | 20      | Max results to return (1–100)      |
| `country` | string | —       | Filter by exact country_code       |

**Success response (200):**
```json
{
  "locations": [
    {
      "id": 1,
      "city": "San Francisco",
      "country": "United States",
      "country_code": "US",
      "latitude": 37.7749,
      "longitude": -122.4194,
      "temperature": 11.3,
      "humidity": 88,
      "wind_speed": 10.9,
      "precipitation": 0.0,
      "condition": "cloudy",
      "fetched_at": "2026-04-07T14:30:00Z"
    }
  ],
  "total": 1
}
```

---

### GET /reports/summary

Aggregate summary across all stored locations.

**Success response (200):**
```json
{
  "total_locations": 12,
  "average_temperature": 18.4,
  "hottest": {
    "city": "Phoenix",
    "temperature": 38.2
  },
  "coldest": {
    "city": "Reykjavik",
    "temperature": -1.3
  },
  "most_common_condition": "cloudy",
  "last_updated": "2026-04-07T14:30:00Z"
}
```

If no locations exist yet, `total_locations` is `0`, `average_temperature` is `null`, `hottest` and `coldest` are `null`, `most_common_condition` is `null`, and `last_updated` is `null`.

---

## Weather Condition Mapping

Each location gets a `condition` derived from the WMO weather code returned by Open-Meteo:

| WMO code(s)     | condition    |
|------------------|-------------|
| 0                | `"clear"`   |
| 1, 2, 3         | `"cloudy"`  |
| 45, 48          | `"fog"`     |
| 51–67           | `"rain"`    |
| 71–77           | `"snow"`    |
| 80–82           | `"rain"`    |
| 85, 86          | `"snow"`    |
| 95, 96, 99      | `"storm"`   |

The `most_common_condition` in the summary report is the condition that appears most frequently across all stored locations.

---

## Data Source

All weather data comes from the Open-Meteo API (`https://open-meteo.com/`). This is a free, public API. No API key is required.

**Geocoding — resolve city name to coordinates:**
```
GET https://geocoding-api.open-meteo.com/v1/search?name=San+Francisco&count=1&language=en&format=json
```

Response:
```json
{
  "results": [
    {
      "name": "San Francisco",
      "latitude": 37.77493,
      "longitude": -122.41942,
      "country_code": "US",
      "country": "United States"
    }
  ]
}
```

If the `results` array is empty or absent, the city was not found.

**Current weather — fetch conditions for coordinates:**
```
GET https://api.open-meteo.com/v1/forecast?latitude=37.7749&longitude=-122.4194&current=temperature_2m,relative_humidity_2m,wind_speed_10m,precipitation,weather_code
```

Response:
```json
{
  "current": {
    "time": "2026-04-07T12:45",
    "temperature_2m": 11.3,
    "relative_humidity_2m": 88,
    "wind_speed_10m": 10.9,
    "precipitation": 0.0,
    "weather_code": 3
  }
}
```
