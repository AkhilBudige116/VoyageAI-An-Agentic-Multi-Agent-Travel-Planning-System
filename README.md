# VoyageAI — Agentic Multi-Agent Travel Planning System

VoyageAI is an AI-powered travel planner built with FastAPI and LangGraph. It takes a natural-language trip request and coordinates specialized agents to gather flight, hotel, and weather information, create an itinerary, and return a consolidated travel plan.

## Features

- Natural-language trip planning through a web interface and JSON API.
- A sequential LangGraph workflow for flight guidance, hotel search, weather, itinerary creation, and final recommendations.
- Groq-hosted Llama 3.3 70B for destination extraction and travel-planning responses.
- MCP integrations for Tavily search, AviationStack flight/airport data, and a custom OpenWeather MCP server.
- PostgreSQL-backed LangGraph checkpoints, keyed by a persistent conversation thread ID.
- A FastAPI health endpoint and automatically generated API documentation.
- Frontend controls for copying a plan and downloading it as a PDF.

Flight prices and availability depend on the connected data sources; treat generated plans and estimates as guidance and verify details with providers before booking.

## How a request is processed

The workflow in `backend.py` runs these LangGraph nodes in order:

1. **Flight agent** — retrieves airport and airline information through AviationStack MCP and uses the LLM to produce route guidance and estimated airfare information.
2. **Hotel agent** — searches for hotel information with Tavily MCP.
3. **Weather agent** — extracts the destination with the LLM and requests current conditions and a forecast from the custom Weather MCP server.
4. **Itinerary agent** — combines the request and gathered results into a practical itinerary.
5. **Final agent** — prepares a structured response with a trip summary, transport and accommodation information, weather, itinerary, budget, and recommendations.

The graph saves conversation state using a PostgreSQL checkpointer. The web client sends the returned `thread_id` with later requests to continue that thread.

## Project structure

```text
.
├── app.py                       # FastAPI app, web routes, and /api/travel endpoint
├── backend.py                   # LangGraph workflow, agents, and PostgreSQL checkpointing
├── mcp_client.py                # Tavily, AviationStack, and Weather MCP client setup
├── custom_weather_mcp_server.py # Local MCP tools backed by OpenWeather
├── tools/
│   ├── flight_tool.py           # Direct AviationStack helper
│   └── tavily_tool.py           # Direct Tavily search helper
├── templates/
│   └── index.html               # Travel planner page template
├── static/
│   ├── style.css                # Page styles
│   └── scripts.js               # Browser-side API, copy, and PDF logic
├── test.py                      # MCP connectivity/tool-list diagnostic
├── mcp_client_test.py           # Tavily-only MCP diagnostic
├── requirements.txt             # Pinned Python dependencies
├── pyproject.toml               # Project metadata
├── .python-version              # Python version used by the project (3.14)
├── Dockerfile                   # Container build and Uvicorn startup
└── LICENSE                      # MIT License
```

`main.py` is a small placeholder and is not the web application entry point. Start the server with `app.py` or Uvicorn as shown below.

## Requirements

- Python 3.14 (the version specified in `.python-version` and `pyproject.toml`).
- PostgreSQL with a connection URL accessible to the application.
- API credentials for Groq, Tavily, AviationStack, and OpenWeather.
- `uvx` on `PATH` for the AviationStack MCP server. It is provided by the `uv` package (`python -m pip install uv`).
- Internet access for the hosted APIs and the Tavily MCP service.

## Configuration

Create a `.env` file in the project root. Do not commit this file or share its values.

```dotenv
GROQ_API_KEY=your_groq_api_key
DATABASE_URL=your_postgresql_connection_url
TAVILY_API_KEY=your_tavily_api_key
AVIATIONSTACK_API_KEY=your_aviationstack_api_key
OPENWEATHER_API_KEY=your_openweather_api_key
```

`DATABASE_URL` must point to a PostgreSQL database. The application adds `sslmode=require` if the URL does not already specify an SSL mode, then initializes the LangGraph checkpoint tables at startup. `GROQ_API_KEY` and `DATABASE_URL` are required while importing the backend; the external search/weather keys are needed for their corresponding MCP integrations.

## Run locally

From the project root in PowerShell:

```powershell
py -3.14 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip install uv
```

Configure `.env`, ensure `uvx` is available in the activated environment's `PATH`, then start the app:

```powershell
python app.py
```

Alternatively, run Uvicorn directly:

```powershell
uvicorn app:app --host 127.0.0.1 --port 8000 --reload
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) for the planner. FastAPI's interactive API documentation is at [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs), and the health check is [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health).

## API

### `GET /health`

Returns an application status response.

### `POST /api/travel`

Request JSON:

```json
{
  "message": "Plan a 7-day trip to Japan from Bangladesh within my budget",
  "thread_id": null
}
```

`message` is required. Pass the `thread_id` returned by a previous response to continue its saved conversation; omit it or send `null` to start a new thread.

Successful responses include `success`, `thread_id`, `answer`, `flight_results`, `hotel_results`, `itinerary`, and `llm_calls`. Invalid empty messages return HTTP 400; processing errors return HTTP 500 with an error message.

## MCP diagnostic

After configuring credentials and installing `uv`, run:

```powershell
python test.py
```

This script asks the configured MCP servers to list their available tools. It is a connectivity diagnostic, not an automated unit-test suite. `mcp_client_test.py` contains a separate Tavily-only diagnostic.

## Docker

The repository includes a Dockerfile that installs `requirements.txt` and starts Uvicorn on port 8000. Build and run:

```powershell
docker build -t voyageai .
docker run --rm -p 8000:8000 --env-file .env voyageai
```

The Dockerfile currently uses Python 3.11, while `.python-version` and `pyproject.toml` specify Python 3.14 for local development. Check dependency/runtime compatibility before choosing a deployment image. Also note that the current `.dockerignore` does not exclude `.env`; do not include real credentials in the Docker build context or image.

## Implementation notes

- PostgreSQL is needed at startup because `backend.py` creates the checkpointer during application initialization.
- The frontend template currently references `/static/script.js`, but the checked-in browser script is named `static/scripts.js`. The browser script will not load until these names are made consistent.
- `tools/flight_tool.py` and `tools/tavily_tool.py` provide direct API helpers; the active graph uses the MCP client functions in `mcp_client.py`.

## License

This project is distributed under the MIT License. See [LICENSE](./LICENSE).
