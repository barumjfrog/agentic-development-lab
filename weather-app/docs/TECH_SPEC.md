# SkyLog — Technical Specification

## Stack

| Component        | Choice                          | Reason                                           |
|------------------|---------------------------------|--------------------------------------------------|
| Language         | Python 3.11+                    | Target runtime                                   |
| Framework        | FastAPI                         | Async-native, auto-generated OpenAPI docs        |
| HTTP client      | httpx                           | Async HTTP for calling Open-Meteo APIs           |
| ORM              | SQLModel                        | Combines SQLAlchemy engine with Pydantic models   |
| Database         | SQLite                          | Zero-config, file-based, sufficient for this scope|
| Server           | Uvicorn                         | Standard ASGI server for FastAPI                  |
| Config           | pydantic-settings               | Typed env var loading with `.env` support         |

## Project Structure

```
weather-app/
├── app/
│   ├── __init__.py
│   ├── main.py                # FastAPI app instance, lifespan hook, CORS middleware
│   ├── config.py              # Settings class (env vars, defaults)
│   ├── database.py            # SQLite engine, session factory, create_db_and_tables()
│   ├── models.py              # SQLModel table definitions
│   ├── schemas.py             # Pydantic request/response models (not DB-bound)
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── locations.py       # POST /locations, POST /locations/import, GET /locations
│   │   └── reports.py         # GET /reports/summary
│   └── services/
│       ├── __init__.py
│       ├── weather_client.py  # Async wrapper around Open-Meteo geocoding + weather APIs
│       └── location_service.py # Orchestrates: parse input → geocode → fetch weather → persist
├── tests/
│   ├── __init__.py
│   ├── test_locations.py      # Tests for /locations endpoints
│   └── test_weather_client.py # Tests for weather client with mocked responses
├── pyproject.toml
├── .env.example
└── README.md
```

## Configuration

Managed via `pydantic-settings`. All values have sensible defaults and can be overridden with environment variables or a `.env` file.

| Variable            | Type   | Default                                                              |
|---------------------|--------|----------------------------------------------------------------------|
| `DATABASE_URL`      | string | `sqlite:///./skylog.db`                                              |
| `GEOCODING_API_URL` | string | `https://geocoding-api.open-meteo.com/v1/search`                     |
| `WEATHER_API_URL`   | string | `https://api.open-meteo.com/v1/forecast`                             |
| `LOG_LEVEL`         | string | `INFO`                                                               |

**`.env.example`:**
```
DATABASE_URL=sqlite:///./skylog.db
GEOCODING_API_URL=https://geocoding-api.open-meteo.com/v1/search
WEATHER_API_URL=https://api.open-meteo.com/v1/forecast
LOG_LEVEL=INFO
```

## Database Schema

Single table: `location`

| Column          | Type     | Constraints                    |
|-----------------|----------|--------------------------------|
| `id`            | integer  | Primary key, auto-increment    |
| `city`          | string   | Indexed, not null              |
| `country`       | string   | Not null                       |
| `country_code`  | string   | Not null                       |
| `latitude`      | float    | Not null                       |
| `longitude`     | float    | Not null                       |
| `temperature`   | float    | Not null                       |
| `humidity`      | integer  | Not null                       |
| `wind_speed`    | float    | Not null                       |
| `precipitation` | float    | Not null, default 0.0          |
| `weather_code`  | integer  | Not null                       |
| `condition`     | string   | Not null (`clear`, `cloudy`, `fog`, `rain`, `snow`, `storm`) |
| `fetched_at`    | datetime | Not null, default UTC now      |

The database file is created automatically on first startup via the lifespan hook.

## Module Responsibilities

### `app/main.py`
- Create the FastAPI instance with metadata (title, version, description).
- Register routers from `app/routers/`.
- Use the FastAPI `lifespan` context manager to call `create_db_and_tables()` on startup.
- Add CORS middleware (allow all origins — this is a local dev tool).

### `app/config.py`
- A single `Settings` class inheriting from `pydantic_settings.BaseSettings`.
- Loads from environment variables and `.env` file.

### `app/database.py`
- Create a SQLModel `engine` from `settings.DATABASE_URL`.
- Provide a `get_session()` dependency that yields a `Session`.
- `create_db_and_tables()` calls `SQLModel.metadata.create_all(engine)`.

### `app/models.py`
- `Location(SQLModel, table=True)` matching the schema above.

### `app/schemas.py`
- `AddLocationRequest`: `city: str`, `country_code: str | None = None`.
- `ImportLocationsRequest`: `content: str`.
- `LocationResponse`: full single-location response shape with weather data.
- `ImportLocationsResponse`: total/added/skipped counts plus results array.
- `LocationListResponse`: paginated list of locations.
- `ReportSummaryResponse`: aggregate summary with hottest/coldest/averages.

### `app/services/weather_client.py`
- `async def geocode(city: str, country_code: str | None = None) -> dict | None`
  - Uses `httpx.AsyncClient` to GET the Open-Meteo geocoding API.
  - Passes `name` and `count=1`. If `country_code` is provided, passes it as well.
  - Returns the first result from the `results` array, or `None` if no match.
  - Raises `httpx.HTTPStatusError` on non-2xx responses (caller handles this).
- `async def fetch_weather(latitude: float, longitude: float) -> dict`
  - Uses `httpx.AsyncClient` to GET the Open-Meteo forecast API.
  - Requests `current=temperature_2m,relative_humidity_2m,wind_speed_10m,precipitation,weather_code`.
  - Returns the `current` object from the response.
  - Raises `httpx.HTTPStatusError` on non-2xx responses (caller handles this).
- Use a shared `httpx.AsyncClient` instance (created once, closed on shutdown).

### `app/services/location_service.py`
- `def map_weather_code(code: int) -> str`
  - Maps a WMO weather code to a condition string using the table from the product spec.
- `async def add_location(city: str, country_code: str | None, session: Session) -> LocationResponse`
  - Calls `weather_client.geocode()`.
  - If no result, raises a 404 error.
  - Calls `weather_client.fetch_weather()` with the resolved coordinates.
  - Maps the weather code to a condition.
  - Persists a `Location` to the database.
  - Returns the structured response.
- `async def import_locations(content: str, session: Session) -> ImportLocationsResponse`
  - Parses the content line by line.
  - Calls `add_location()` for each valid line.
  - Aggregates results, collects skipped lines with reasons.
- `def parse_location_line(line: str) -> tuple[str, str | None] | None`
  - Extracts city name and optional country_code from a single line.
  - Returns `None` for blank lines and comments.

## Error Handling

- If the Open-Meteo API returns a non-2xx status or is unreachable, the endpoint returns **503** with `{"detail": "Weather service is temporarily unavailable. Try again later."}`.
- If a city cannot be geocoded, the endpoint returns **404** with `{"detail": "City not found. Check the spelling or add a country_code to narrow the search."}`.
- Input validation errors use FastAPI's built-in 422 response.
- Database errors return **500** with `{"detail": "Internal server error."}`.
- Log all errors at `ERROR` level with the exception details.

## Testing

Use `pytest` with `httpx.AsyncClient` for endpoint tests.

- `test_locations.py`: Test the `/locations` POST endpoint with a mocked geocoding and weather response (both success and city-not-found cases). Test `/locations/import` with multi-line input including comments and blank lines. Test `GET /locations` with and without the `country` filter.
- `test_weather_client.py`: Test the weather client with `respx` or `pytest-httpx` to mock external HTTP calls.

Use an in-memory SQLite database for tests (`sqlite://`).

## How to Run

```bash
pip install .            # install runtime dependencies
pip install ".[dev]"     # install with dev/test dependencies
uvicorn app.main:app --reload
```

The API is then available at `http://localhost:8000`. Interactive docs at `http://localhost:8000/docs`.

## Building

The project must be buildable into a wheel via `python -m build --wheel`. Use `pyproject.toml` with setuptools as the build backend. The package name is `weather-app`, version `0.1.0`. All dependency management goes through `pyproject.toml` — no `requirements.txt`.

Because the project uses a flat layout (the `app/` package sits at the repo root alongside other directories like virtual environments and build artifacts), explicit package discovery is required to prevent setuptools from refusing to build:

```toml
[tool.setuptools.packages.find]
where = ["."]
include = ["app*"]
exclude = ["venvdir*", "build*", "dist*", "tests*", "*.egg-info*"]
```

## Dependencies

All dependencies are declared in `pyproject.toml`.

Runtime (`[project].dependencies`): `fastapi==0.109.0`, `uvicorn`, `sqlmodel`, `httpx`, `pydantic-settings`, `python-dotenv`. Pin `fastapi==0.109.0` — the application was developed against this version and has not been validated against newer releases.

Dev/test (`[project.optional-dependencies].dev`): `pytest`, `pytest-asyncio`. `httpx` is also used as the test client via `ASGITransport`.
